# Relaix

A self-hosted SMS gateway: your Android phones become SMS-sending nodes, managed by a lightweight Go control plane over gRPC.

## Status

**Early development — control plane runs, no SMS leaves a phone yet.**

The Go control plane is wired together and works end to end: `docker compose up --build`
brings up Postgres and `gatewayd`, a device enrolls, the scheduler pushes a job down the
stream and the result comes back. It has only ever been exercised against a throwaway gRPC
client, because the Android agent is still a Compose scaffold — it does not enroll, connect,
or send. That is the work in progress. The implementation lands one atomic commit at a time;
the git history is the record of what exists so far.

## Architecture

Relaix borrows the shape of the Kubernetes control plane. A Go server holds the registry
of known devices, receives heartbeats, and schedules work; each Android phone runs an agent
that dials out and holds an open gRPC bidirectional stream. Because phones sit behind
carrier NAT and are never reachable from the internet, the device always initiates the
connection and the server pushes jobs down the stream it already has.

```
   HTTP client                Relaix control plane (Go)              Android agents
        │                                                                  │
        │  POST /send          ┌──────────────────────────┐                │
        ├─────────────────────►│  REST API                │                │
        │                      ├──────────────────────────┤                │
        │                      │  scheduler (tick loop)   │                │
        │                      ├──────────────────────────┤   gRPC bidi    │
        │                      │  hub (actor goroutine)   │◄══════════════►│ phone A
        │                      ├──────────────────────────┤   stream       │
        │                      │  gRPC server             │◄══════════════►│ phone B
        │                      ├──────────────────────────┤                │
        │                      │  callback watcher        │                │◄─ SMS out
        │  webhook (HMAC)      ├──────────────────────────┤                │
        │◄─────────────────────┤  Postgres                │                │
        │                      └──────────────────────────┘                │
```

Full write-up, including the gRPC-vs-WebSocket decision and the known limitations:
**[docs/architecture.md](docs/architecture.md)**.

## Repository structure

| Path | Contents |
| --- | --- |
| `server/` | Go control plane — submodule → [relaix-server](https://github.com/KGMA74/relaix-server). |
| `android/` | Android agent — submodule → [relaix-agent](https://github.com/KGMA74/relaix-agent). |
| `proto/` | Protobuf contract shared by server and agent (`proto/smsgateway/v1/`). |
| `buf.yaml`, `buf.gen.yaml` | Lint and codegen for the contract. |
| `docs/` | Architecture and wire protocol. |
| `docker-compose.yml` | Local development stack. Placeholder for now. |

Clone with the submodule:

```sh
git clone --recurse-submodules https://github.com/KGMA74/relaix.git
```

This repository holds **no Go module**. `proto/` is the single source of truth for the
contract, and `buf.gen.yaml` writes the generated Go into `server/`, where it is committed
in `relaix-server`. Each consumer repository owns the code generated from the contract:

```sh
buf lint && buf generate   # from this directory
```

## Docs

- [Architecture](docs/architecture.md) — components, design decisions, and their rationale.
- [Protocol](docs/protocol.md) — the gRPC service and every message on the wire.

## License

[Apache License 2.0](LICENSE).
