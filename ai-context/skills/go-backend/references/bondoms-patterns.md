# BondOMS Patterns (Fiber + gRPC, flat layout)

`BondOMS/` (in the `c:\_project_git` bond monorepo) — Go, Fiber v2, gRPC client, **dual DB**, flat layout. Core OMS for HSC private bond (Bond Riêng Lẻ / TPRL) trading. Carbon-OMS was forked from this codebase, so the shape is familiar — but BondOMS still has live FIX/NATS/gRPC integrations Carbon-OMS dropped.

When adding to BondOMS, **match this flat style** — do not introduce clean-arch layering.

## Stack & layout

- Go (1.23-era monorepo; verify `go.mod`), Fiber v2, gRPC client to `BondFIXOrderGW`.
- Flat layout: `main.go`, `router.go`, `handler-api.go` (6000+ line monolith), `handler-common.go`, `handler-core-api.go`, `handler-quote-*.go`, `retryQueryResult.go`, `datastruct/`, `infra/http/vcb/`.
- **Dual DB**: `dbpg` (Postgres — owns `order`, `logs`, `logs_hnx`) + `dbG3SB` (MSSQL/sqlserver — **read-only** mirror of G3SB).
- Logging: stdlib `log.Printf` + a thin `logs` wrapper (`logs.Warnf`/`logs.Errorf`). **No tests, no sentry/slack/teams alerting** — `logs.Errorf` is the only signal an operator gets.
- Shares `HscGoModules` (Go 1.24 shared lib) with HSC_STP and BondFIXOrderGW — see `hscgomodules.md`.

## Routing (`router.go`)

`SetupRoutes(app)`, routes grouped under `app.Group("/equity")`. Handlers split across `handler-*.go` files. Key routes:

| Path | Handler | File |
|---|---|---|
| POST `/equity/order/new` | NewOrderHandler | handler-api.go |
| POST `/equity/order/amend` | AmendOrderHandler | handler-api.go |
| POST `/equity/order/cancel` | CancelOrderHandler | handler-api.go |
| POST `/equity/quote/new` | PlaceQuoteHandler | handler-quote-place.go |
| POST `/equity/vcb/deposit` | DepositVCB | handler-api.go |

Line numbers drift — grep the handler name. Fiber handler signature `(c *fiber.Ctx) error`, same as Carbon-OMS.

## Background workers / crons

Registered in `retryQueryResult.go` + `main.go`. Single-instance via `cron.SkipIfStillRunning`.

| Job | Schedule | Function |
|---|---|---|
| `RetryUpdateHnx` | `ReadStpReceiveCronExp` (~15s) | the **real order state machine** — polls `logs_hnx`, applies status transitions |
| `RetryBankDeposit` | `ReadStpReceiveCronExp` (~15s) | VCB bank-transfer polling |
| `RetrySubscribe` | env `NATS_RETRY_HOUR`/`MIN` | NATS reconnect |
| `getFeeMaster` | `CronSyncFeeMaster` | fee-master sync |

`SubscribeNatStream` (in `handler-api.go`, spawned as a background goroutine from `main.go`) is the **real FIX ingress** — NATS JetStream consumer keyed on `data["MsgType"]`. `handleOne` only logs; `RetryUpdateHnx` does the actual state transitions.

## Database access — raw SQL + goqu

Unlike Carbon-OMS (where `goqu` is in `go.mod` but unused), BondOMS **actively uses `goqu`** in places (`goqu.From("order").Select(...)` then `Scan()`). Both raw `database/sql` and `goqu` query-builder coexist.

**SELECT/Scan field-alignment trap (BondOMS-specific):** when adding logic that reads a new `DataUpdate.X` field after a `Scan()`, the Go compiler will NOT catch a missing field — the struct field defaults to its zero value (empty string), causing a silent runtime lookup failure (0 rows). Real bug: cross-firm BOND amend ACK in `RetryUpdateHnx` added `OrderIdRoot`/`OrderCompany`/`ReciprocalAccount` reads but the upstream `SELECT` still had only 8 old fields → `OrderIdRoot=""` → over-hold, hidden 9 days in UAT.

**Mandatory checklist** when touching `RetryUpdateHnx`, `updateOrderFromHnx`, `updateOrderFromPartnerHnx`, `AmendOrderHandler`, or any `goqu ... Select() + Scan()` block:
- Every field read after `Scan()` must appear in BOTH `.Select(...)` AND `.Scan(&...)`.
- `Scan()` arg order must match `Select()` order (positional).
- Copying logic between branches (`!BOND` → `BOND`): copy the matching SELECT/Scan too, not just the logic.
- WHERE that can match root + edit rows: filter `order_type='order_root'` or `ORDER BY id ... LIMIT 1` for determinism.
- Fail path: `logs.Errorf` (alert), never bare `log.Printf` (silent).

`defer rows.Close()` nil-panic: place `defer rows.Close()` **after** the `err != nil` check — `dbpg.Query` returns nil `rows` on error; deferring before the check panics on DB EOF / pool exhaustion.

## G3SB SOAP client

SOAP to G3SB for asset hold/release and pre-trade checks. Same `messageTransfer`/CDATA envelope family as Carbon — see the `g3-core-integration` skill for the transport. **Never call G3 inside a Postgres transaction.** Hold(new)-before-release(old) ordering.

## Status enums

`hnx_status` (1-15) and `vsd_status` (1-10) live in `datastruct/constant.go` and are **two parallel independent state machines** — never merge. ~40 sites in `handler-api.go` assign status values directly; there is no `transitionTo()` guard helper, so adding one means auditing all 40.

## Known anti-patterns / risks (flag in review)

| ID | Issue | Fix |
|---|---|---|
| R1 | `BondFIXOrderGW/internal/client/in_msg_hdl.go` `OnFIX44*` stubs are dead code — real path is NATS | Don't "fix" the stubs; state machine is `SubscribeNatStream` |
| R2 | `order.vsd_status` has two writers (BondOMS + BondTradingMiddleware) → race | BondOMS should be sole owner per spec |
| R4 | No transition guard for hnx/vsd_status — ~40 direct assignments | Adding a guard = audit all sites |
| R13 | NATS consumer throttled to ~1 msg/s (`natsHandleMu` + `time.Sleep(1s)`) | HNX bursts dozens/sec — investigate the underlying `insertLogHnx` race before removing the sleep |
| R14 | JetStream published but consumed as core-NATS (`nc.ChanSubscribe`) — messages lost during downtime, `DLQ.*` orphaned | Use `js.PullSubscribe` + durable name |
| R16 | `AmendOrderHandler` trusts client-supplied `payload.HNXStatus` (no auth) — #1 security hole | Server-derive status; never trust client status |
| R17 | Cancel state machine dead — enums 10/11/13/14 never set | Greenfield work, not a one-liner |
| R20 | 17+ `defer rows.Close()` nil-panic sites remain | Move `defer` after the err check |

Also: same Carbon-OMS-class issues apply if present — string-concat SQL (A1), `InsecureSkipVerify` (A2), hardcoded secrets (A3), external-call-in-tx (A4), silent error handling (A6). See `anti-patterns.md`.

See `anti-patterns.md` for the cross-service catalog, `hscgomodules.md` for shared code, `carbon-oms-patterns.md` for the forked-from sibling (Carbon-OMS dropped the FIX/NATS/gRPC integrations BondOMS keeps).