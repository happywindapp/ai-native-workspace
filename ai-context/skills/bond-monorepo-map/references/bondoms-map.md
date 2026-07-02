# BondOMS Map — HTTP Routes & Cron Jobs

Core OMS. Go / Fiber / gRPC. Routes registered in `router.go`; handlers split across `handler-*.go` files.

> As-of: built 2026-04-17, refreshed 2026-05-14. Line numbers drift — grep the handler name for the canonical location.

## HTTP Routes — query endpoints (read-only)

| Path | Handler | File:Line |
|---|---|---|
| GET `/` | HealthCheck | handler-common.go:10 |
| GET `/equity/fee/:accountNo` | EquityFeeHandler | handler-common.go:50 |
| GET `/equity/info/:accountNo` | EquityAccountInfoHandler | handler-common.go:104 |
| GET `/equity/portfolio/:accountNo` | EquityPortHandler | handler-common.go:67 |
| GET `/equity/cash/:accountNo` | EquityCashHandler | handler-common.go:185 |
| GET `/equity/order/summary` | SummaryHandler | handler-common.go:209 |
| GET `/equity/order/comparison` | DailyComparisonHandler | handler-common.go:258 |
| GET `/equity/order/list` | ListOrderHandler | handler-common.go:339 |
| POST `/equity/order/list-status` | ListStatusOrderHandler | handler-common.go:587 |

## HTTP Routes — order lifecycle

| Path | Handler | File:Line |
|---|---|---|
| POST `/equity/order/new` | NewOrderHandler | handler-api.go:648 |
| POST `/equity/order/amend` | AmendOrderHandler | handler-api.go:1064 |
| POST `/equity/order/match` | MatchOrderHnxHandler | handler-api.go:2299 |
| POST `/equity/order/accept` | AcceptOrderHnxHandler | handler-api.go:2494 |
| POST `/equity/order/cancel` | CancelOrderHandler | handler-api.go:2679 |
| POST `/equity/order/edit` | EditOrderHandler | handler-api.go:2899 |
| POST `/equity/order/update-history` | UpdateHistoryOrderHandler | handler-api.go:979 |

## HTTP Routes — TTDT quote endpoints (added 2026-05)

| Path | Handler | File |
|---|---|---|
| POST `/equity/quote/new` | PlaceQuoteHandler | handler-quote-place.go |
| POST `/equity/quote/amend` | AmendQuoteHandler | handler-quote-amend-cancel.go |
| POST `/equity/quote/cancel` | CancelQuoteHandler | handler-quote-cancel.go |
| POST `/equity/quote/accept` | AcceptQuoteHandler | handler-quote-accept.go |

Full TTDT detail → `ttdt-quote-map.md`.

## HTTP Routes — VCB bank endpoints

| Path | Handler | File:Line |
|---|---|---|
| POST `/equity/vcb/deposit` | DepositVCB | handler-api.go:2958 |
| POST `/equity/vcb/withdraw` | WithdrawVCB | handler-api.go:2991 |
| GET `/equity/vcb/balance/:account` | GetBalance | handler-api.go:3029 |
| GET `/equity/vcb/session` | GetSessionLoginVCB | handler-api.go:3054 |

## Cron Jobs / background workers

| Job | Schedule | Function | File:Line |
|---|---|---|---|
| Fee sync | `CronSyncFeeMaster` | getFeeMaster | handler-core-api.go:318 |
| Bank poll | `ReadStpReceiveCronExp` | RetryBankDeposit | handler-api.go:3572 |
| HNX retry (state machine) | `ReadStpReceiveCronExp` (~15s) | RetryUpdateHnx | handler-api.go:3643 |
| NATS reconnect | env `NATS_RETRY_HOUR`/`NATS_RETRY_MIN` | RetrySubscribe | handler-api.go:6465 |

- `RetryBankDeposit` and `RetryUpdateHnx` are registered in `retryQueryResult.go` (~`:18` / `:23`), both using `datastruct.ReadStpReceiveCronExp`.
- `RetrySubscribe` is started from `main.go` (~`:193`).
- `RetryUpdateHnx` is the real order state machine — single-instance via `cron.SkipIfStillRunning`. `handleOne` only logs; the cron does the actual state transitions.

## VCB bank integration (`infra/http/vcb/vcb.go`)

| Method | Line | Purpose |
|---|---|---|
| `login()` | :77 | Auth VCB API |
| `RefreshSession()` | :123 | Refresh session token |
| `Transfer()` | :180 | Send money transfer |
| `GetTransactionStatus()` | :298 | Poll transfer status |
| `GetBalance()` | :254 | Account balance |
| `GetReportTransaction()` | :342 | Transaction report |
| `GetReportAvailableBalance()` | :377 | Balance report |
| `GetReportStpDepositWithdraw()` | :412 | STP deposit/withdrawal report |

### VCB payment flow (where it runs)

Triggered inside `AmendOrderHandler` when both `vsdStatus=WaitingForConfirm` AND `bankStatus=WaitingForConfirm` (~`:1246-1261`):
1. `callTransferToVCB()` (~`:2999`) → `vcbInstance.Transfer()`.
2. `RetryBankDeposit()` polls every 15s → `vcbInstance.GetTransactionStatus()`.
3. Success → `bankStatus=Success(4)`, `updateStatusOrder()`.
4. Fail → `bankStatus=Failed(5)`.

## G3SB query functions (`handler-common.go`)

Direct SQL on `dbG3SB` (MSSQL, read-only mirror):
`portQueryString`, `portStockQueryString`, `clientQueryString`, `clientNameQueryString`, `clientIDQueryString`, `clientRegistrationTypeQueryString`, `clientContactQueryString`, `cashQueryString`, `accountContractString`, `getFeeQueryString`, `feeMasterQueryString`.

## Constants

- `hnx_status` / `vsd_status` enums → `datastruct/constant.go:5-37`.
- Payment type: `Payment_Type_Now = 1` (T+0 immediate) · `Payment_Type_TT = 2` (delayed/future).
- TTDT transaction types: `Transaction_Type_TTDT = "outright_ttdt"` (~`:88`) · `Transaction_Type_BCGD = "outright_bcgd"` (~`:87`).