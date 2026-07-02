# Service Roles & Entry Points

Reference map for a typical microservice trading platform. Adjust names to the actual repos in the workspace.

## Service responsibilities

| Service | Role | Owns |
|---|---|---|
| Terminal FE | Trader-facing UI; builds order requests | UI state, request payloads |
| Admin FE (STP Admin) | Ops/admin UI | Admin actions, monitoring |
| Middleware | API gateway / orchestration; auth, validation, routing | Session, request fan-out |
| OMS (Order Management System) | Order lifecycle authority; state machine; risk/hold/release | Order records, order state |
| FIX / Integration Gateway | Protocol bridge to exchange | FIX sessions, message correlation |
| STP (Straight-Through Processing) | Post-trade processing, settlement, confirmations | Trade/settlement records |
| Shared modules (Go/Common libs) | Cross-service models, MQ helpers, constants | DTOs, subjects, enums |

## Who talks to whom

```
Terminal FE ──REST──> Middleware ──REST/gRPC──> OMS
Admin FE   ──REST──> STP
OMS  <──MQ──>  FIX Gateway  <──FIX──>  Exchange
OMS  <──MQ──>  STP
Gateway ──MQ──> OMS (execution reports, status)
```

## Entry-point folder conventions

When locating the handler for an operation, grep these per service:

| Service type | Sync entry | Async entry |
|---|---|---|
| Middleware / API | `route/`, `routes/`, `controller/`, `controllers/` | — |
| OMS | `controller/`, `handler/`, `service/` | `consumer/`, `subscriber/`, `nats/`, `events/` |
| Gateway | `handler/`, `session/`, `fix/` | `consumer/`, `publisher/` |
| STP | `service/`, `processor/` | `consumer/`, `worker/` |

## Locating an operation handler

1. Grep for the operation verb: `amend`, `modify`, `cancel`, `hold`, `release`, `match`, `accept`, `newOrder`, `placeOrder`.
2. Grep for the route path or message subject (e.g. `order.amend`, `/orders/:id/cancel`).
3. Grep for the FIX message type constant: `35=D`, `35=F`, `35=G`, `35=8`, or symbolic names `NewOrderSingle`, `OrderCancelRequest`, `OrderCancelReplaceRequest`, `ExecutionReport`.
4. Cross-reference shared-module constants for subject names and enum values.

## Per-flow checklist

For each operation, confirm you found:
- [ ] FE request builder
- [ ] Middleware route + validation
- [ ] OMS sync handler + state write
- [ ] OMS MQ publish
- [ ] Gateway MQ consumer
- [ ] Gateway FIX send
- [ ] Gateway FIX-response handler + MQ publish back
- [ ] OMS MQ consumer for status + state update
- [ ] STP consumer (for match/fill flows)