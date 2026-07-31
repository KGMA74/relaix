# Architecture

Relaix turns a fleet of ordinary Android phones into SMS-sending nodes driven by a single
Go control plane. The design deliberately mirrors the Kubernetes control plane: nodes
register themselves, send heartbeats, and a scheduler assigns work to whichever node is
eligible.

This document explains the components and, for each significant choice, why it was made
that way. The wire format itself lives in [protocol.md](protocol.md).

---

## 1. Overview

Two halves:

- **Control plane** — a Go server (`gatewayd`). Exposes a REST API to callers, a gRPC
  endpoint to devices, and owns the Postgres database. Stateless with respect to SMS
  content; authoritative with respect to job state.
- **Devices** — an Android app (the *agent*) running as a foreground service on each phone.
  It holds one long-lived gRPC stream to the control plane, reports its health, and sends
  SMS via the platform `SmsManager`.

```
       caller                        control plane                    device fleet
  ┌──────────────┐            ┌───────────────────────────┐
  │ your backend │            │  REST API                 │
  └──────┬───────┘            │    POST   /send           │
         │ POST /send         │    GET    /jobs/:id        │
         ├───────────────────►│    DELETE /jobs/:id        │
         │  202 + jobId       │    GET    /devices         │
         │◄───────────────────│    POST   /admin/...       │
         │                    └─────────────┬─────────────┘
         │                                  │ enqueue
         │                    ┌─────────────▼─────────────┐
         │                    │  scheduler (tick loop)    │
         │                    │   pick job → pick device  │
         │                    └─────────────┬─────────────┘
         │                                  │ SendJob(deviceID, job)
         │                    ┌─────────────▼─────────────┐
         │                    │  hub (single goroutine)   │
         │                    │   deviceID → stream chan  │
         │                    └─────────────┬─────────────┘
         │                                  │
         │                    ┌─────────────▼─────────────┐   gRPC bidi stream
         │                    │  gRPC server              │◄══════════════════► agent
         │                    │   Connect / Enroll        │                      │
         │                    └─────────────┬─────────────┘                 SmsManager
         │                                  │ job result                         │
         │                    ┌─────────────▼─────────────┐                      ▼
         │  webhook (HMAC)    │  callback watcher         │                   recipient
         │◄───────────────────│   poll + backoff          │
         │                    └─────────────┬─────────────┘
                              ┌─────────────▼─────────────┐
                              │  Postgres                 │
                              └───────────────────────────┘
```

---

## 2. The NAT constraint

This is the constraint everything else bends around.

Phones on mobile data sit behind carrier-grade NAT. They have no stable, publicly routable
address, no inbound port, and no way to accept a connection. Even on Wi-Fi, a phone's
address changes as it moves between networks and its radio sleeps aggressively. **The
server can never dial a device.**

So the direction is inverted: the device dials out, authenticates, and keeps a single
connection open. Every piece of work the server wants done travels down a channel the
device itself opened. This rules out the obvious alternatives:

- *Server → device HTTP push* — impossible, no reachable address.
- *Device polls a REST endpoint* — works, but adds latency proportional to the poll
  interval and burns battery and mobile data on empty polls. A phone that polls every 5s to
  stay responsive spends most of its radio wake-ups learning there is nothing to do.
- *Push notifications (FCM)* — introduces a third-party dependency into a self-hosted
  product, and FCM makes no delivery-latency guarantee, which is exactly what a gateway
  needs.

A held-open bidirectional stream gives push latency with one connection's worth of
keepalive traffic.

---

## 3. Why gRPC and not WebSocket

Both give a bidirectional byte pipe over one connection, so the choice comes down to what
sits on top:

| | gRPC | WebSocket |
| --- | --- | --- |
| Contract | `.proto` file, single source of truth | ad-hoc JSON, documented by convention |
| Codegen | Go **and** Kotlin from the same schema | hand-written types on both sides, kept in sync manually |
| Framing | built in | you write it |
| Keepalive / reconnect backoff | built in, configurable | you write it |
| Payload size | binary protobuf, compact | JSON text, larger over mobile data |
| Deadlines, status codes, interceptors | standard | you write them |

The decisive point is the two-language contract. Server is Go, agent is Kotlin, and the
message set will change often during development. With a `.proto`, a field added on one
side fails to compile on the other until it is handled — the schema is enforced by the
toolchain. With JSON over WS, a mismatch is a runtime bug on a phone in someone's pocket,
which is the worst place to discover it.

The costs are accepted knowingly: gRPC needs HTTP/2 end to end (so any reverse proxy in
front must speak it), and the generated code is bulkier than a JSON parser. Neither
outweighs a typed, machine-checked contract.

---

## 4. The hub — actor pattern

The hub is the in-memory registry of connected devices: which are online, when each was
last heard from, their reported health, and the channel to write to in order to reach each
one.

It is **a single goroutine owning all of that state**, reachable only by sending it a
command over a channel. Nothing else touches the map. The commands are:

| Command | Meaning |
| --- | --- |
| `register` | a stream opened and authenticated; add the device |
| `unregister` | the stream closed or errored; drop the device |
| `heartbeat` | refresh last-seen and health for a device |
| `sendjob` | hand a job to a specific device's outbound channel |
| `get` | look up one device's current state |
| `listready` | list devices eligible to receive work |

Why an actor rather than a `map` behind a `sync.RWMutex`:

- Concurrent access is inherent here — every gRPC stream is its own goroutine, plus the
  scheduler ticking, plus API handlers reading `/devices`. A mutex makes each *operation*
  safe but not each *decision*: "find a ready device and give it this job" is a read
  followed by a write that must not interleave with another scheduler pass, so it wants a
  transaction, not two locked accesses.
- With one owning goroutine, that composite operation is just sequential code. No lock
  ordering to reason about, no chance of a data race, and the serialization point is
  explicit rather than implied.
- It gives one natural place to observe the system: queue depth on the command channel is a
  direct signal of control-plane pressure.

The trade-off is that every hub operation is serialized through one goroutine, so a slow
handler stalls all of them. The rule is therefore that **no hub command may block** — a
send to a device's outbound channel is non-blocking with a bounded buffer, and a device
whose buffer is full is treated as unhealthy rather than allowed to back up the hub.

**Known limitation:** this state is per-process and in-memory. Two `gatewayd` instances have
two disjoint views of the fleet, and an instance cannot reach a device connected to its
peer. Single instance is the supported topology for V1; see §9.

---

## 5. The scheduler

A tick loop. On each tick it pulls schedulable jobs from the database, asks the hub which
devices are ready, pairs them, and hands each pair to the hub for delivery.

**Device selection** is either explicit or automatic:

- *Explicit* — the caller passed a `deviceId` on `POST /send`. That device is used or the
  job waits (or fails, per mode below). It is never silently rerouted: a caller naming a
  device usually means a specific SIM, a specific number, or a specific carrier, and
  substituting another would be wrong in a way the caller cannot detect.
- *Automatic* — the scheduler picks among ready devices. "Ready" means connected, recently
  heartbeated, and reporting usable health (see the health fields in
  [protocol.md](protocol.md)). Among those it spreads load rather than always choosing the
  same phone, since per-device throughput is the binding constraint.

**Priority** orders the job queue: higher priority jobs are considered first on each tick.
It is ordering, not preemption — a job already handed to a device is not recalled because
something more urgent arrived.

**Modes** decide what happens when no device is available:

- `immediate` — **fail fast**. If nothing can take the job right now, it is rejected and the
  caller is told immediately. For interactive flows (an OTP during a login) a message
  delivered four minutes late is worse than a clean error, because the caller can fall back
  to another channel only if it learns quickly.
- `queued` — the job is durably stored and retried on subsequent ticks until a device
  becomes available or it expires. For bulk or non-interactive traffic, where eventual
  delivery is the goal and a transient empty fleet should not lose the message.

**`scheduledAt`** holds a job back until a wall-clock time. The scheduler simply does not
consider jobs whose `scheduledAt` is in the future; when the time passes they enter the
normal queue and follow the normal selection path. Scheduled jobs are always queued in
spirit — `immediate` semantics apply at the moment the job becomes eligible, not at the
moment it was submitted.

Because job state lives in Postgres and only the assignment decision is in memory, a
scheduler restart loses no work: unacknowledged jobs are simply reconsidered on the next
tick. This makes at-least-once delivery the natural failure mode, which is why the agent
deduplicates by job ID (see [protocol.md](protocol.md)).

---

## 6. Callbacks — notifier and watcher

Callers do not poll for the fate of every message. When a job reaches a terminal state, the
control plane POSTs the outcome to the caller's callback URL.

**Signing (HMAC).** Each callback body is signed with HMAC-SHA256 using a shared secret,
and the signature travels in a header along with a timestamp. The receiver recomputes the
MAC over the raw body and compares in constant time. This gives the receiver two things:
authenticity — the callback really came from your gateway and not from anyone who guessed
the endpoint URL — and integrity. The timestamp is included in the signed material and
receivers should reject stale ones, which bounds replay. HMAC rather than a bearer token
because a token in a header is replayable verbatim by anyone who logs it, and rather than
asymmetric signatures because both ends are operated by the same person and key
distribution is a non-problem here.

**The watcher** exists because the caller's endpoint will be down sometimes. Delivery is
not fire-and-forget: pending callbacks are persisted, and a watcher polls for those not yet
acknowledged and retries them with **exponential backoff** — each failure roughly doubles
the wait, up to a ceiling, with a bounded number of attempts before the callback is marked
failed and left for inspection. Backoff rather than a fixed interval so that a receiver
that is down for an hour is not hammered thousands of times, and so that a fleet-wide
outage does not produce a synchronized retry stampede when the receiver comes back.

Consequence for callers: **callbacks are at-least-once**. The same job outcome may arrive
twice if an acknowledgement is lost. Handle them idempotently, keyed by job ID.

---

## 7. Enrollment

A phone must not be able to join the fleet just by knowing the server address.

1. An operator calls `POST /admin/devices/enroll-token`. The server mints a **single-use,
   short-lived token** and returns it, along with a **QR code** image encoding the token and
   the server endpoint.
2. The operator shows the QR to the phone; the agent scans it. This is the whole reason for
   the QR: it avoids typing a long random string on a phone keyboard, and it carries the
   endpoint alongside the token so there is no separate manual configuration step.
3. The agent calls the unary `Enroll` RPC with the token and its device information. The
   server **consumes the token atomically** — the same token can never enroll a second
   device, so a photograph or screenshot of the QR is worthless after first use.
4. The server creates the device record and returns a long-lived `device_token`. The agent
   stores it and uses it thereafter to open the `Connect` stream.

Enrollment is unary rather than the first message of the stream because it is a distinct,
one-time, non-streaming operation with different auth (a short-lived enrollment token
rather than a device token); keeping it separate keeps the stream's authentication rule
uniform — every message on `Connect` carries a `device_token`, no exceptions.

---

## 8. Data model

Four tables, created by migration `0001_init`:

| Table | Holds |
| --- | --- |
| `devices` | one row per enrolled phone: id, label, phone number, `device_token` (hashed), last-seen, last reported health, enabled flag |
| `jobs` | one row per SMS request: recipient, body, mode, priority, `scheduled_at`, requested `device_id` (nullable), assigned `device_id`, status, callback URL, timestamps |
| `enrollment_tokens` | minted tokens: value (hashed), expiry, consumed-at, resulting `device_id` |
| `job_events` | append-only audit trail of every job state transition, with timestamp and reason |

`job_events` is separate from `jobs` deliberately: `jobs` carries current state and is
updated in place, while the event log is never mutated. That gives an auditable history of
why a message ended up where it did — which device took it, when it was retried, what the
failure was — without complicating the row the scheduler reads on every tick.

Tokens are stored hashed, never in plaintext, so a database dump does not hand over the
fleet.

---

## 9. Multi-instance (V2)

The hub's device map is per process, which caps V1 at a single `gatewayd`. Scaling out
needs two additions, both deferred:

- a **Redis registry** mapping `deviceID → instanceID`, so any instance can find out where a
  device is connected;
- **Redis pub/sub** to forward a job from the instance that scheduled it to the instance
  holding that device's stream.

The hub's command interface is designed to make this a substitution rather than a rewrite:
`sendjob` either writes to a local channel or publishes to the owning instance, and callers
of the hub do not learn which happened.

---

## 10. Known limitations

**No custom sender ID.** Messages arrive from the phone's own MSISDN — the number of the SIM
in the device that sent them. There is no way to present a branded alphanumeric sender
("ACME") or an arbitrary number. This is not a gap in the implementation: alphanumeric
sender IDs are a carrier-network feature provisioned through operators and A2P messaging
agreements, and `SmsManager` on a consumer handset has no API to set the originating
address. Anyone who needs a branded sender needs a carrier or an A2P aggregator, not this
gateway. The corollary is that replies come back to the phone, which is useful — inbound
handling is a natural future feature that a real aggregator makes harder.

**Throughput is bound by the carrier, not the software.** Each phone is a consumer SIM
subject to per-hour and per-day sending limits and to anti-spam heuristics. Scaling means
adding phones and SIMs, not tuning the server. Push a single SIM too hard and the carrier
will suspend it.

**Single control-plane instance in V1.** See §9.

**SMS only.** No MMS, no inbound message handling, no delivery-report reconciliation beyond
what the platform reports.

**At-least-once, not exactly-once.** Both job delivery to a device and callback delivery to
the caller can duplicate under failure. Both are keyed by job ID so both are safe to
deduplicate, but callers must actually do it.
