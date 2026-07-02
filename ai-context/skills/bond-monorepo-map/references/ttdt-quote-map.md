# TTDT Quote Map — BondOMS Negotiated-Trade (Thỏa thuận điện tử) Outright

ĐTTTT (Thỏa thuận điện tử) Outright Quote handling inside BondOMS. Distinct from BCGD (NewOrderCross 35=s/t/u). FIX spec: HNX FIX v1.15 §3.3.6-3.3.9 (format details → `financial-messaging`).

> As-of: Phase-02 routes/handlers/NATS DONE; Phase-03/04 partly TBD (2026-05). Grep handler names to confirm.

## Routes (`router.go` ~`:29-34`)

| Endpoint | Handler | FIX out | Spec § |
|---|---|---|---|
| POST `/equity/quote/new` | PlaceQuoteHandler | Quote 35=S | 3.3.6 |
| POST `/equity/quote/amend` | AmendQuoteHandler | QuoteRequest 35=R | 3.3.7 |
| POST `/equity/quote/cancel` | CancelQuoteHandler | QuoteCancel 35=Z (298=4) | 3.3.8 |
| POST `/equity/quote/accept` | AcceptQuoteHandler | QuoteResponse 35=AJ (694=1) | 3.3.9 |

## Handler files (7 files, ~1575 LOC)

| File | Entry | Responsibility |
|---|---|---|
| `handler-quote-place.go` (~:22) | PlaceQuoteHandler | validate → `validateAccountFormat` → `checkAccountOrder` → `ValidateSymbol` (gRPC) → `holdAssetOrder` → INSERT `order` (order_kind=Order_Root, transaction_type=TTDT) → `SendQuote` → `markQuoteQueued`. Rollback: `markQuoteRejected` + `releaseAssetOrder`. |
| `handler-quote-amend-cancel.go` (~:21) | AmendQuoteHandler | hold(new) BEFORE release(old). INSERT `order_kind=Order_Edit, related_id=old.id`. Fallback fields from old quote. `cleanupG3` rolls back hold-swap on insert/commit fail. |
| `handler-quote-cancel.go` (~:23) | CancelQuoteHandler | Sets `hnx_status=Pending_Cancel` only — G3 hold NOT released here; deferred to NATS consumer on 537=3 ack. Rollback hnx_status on gateway fail. |
| `handler-quote-accept.go` (~:22) | AcceptQuoteHandler | Looks up peer quote via `lookupQuoteByHnxID` (must be hnx_status=Queue). Self-accept rejected. Price/qty must match orig. Acceptor takes OPPOSITE side. INSERT new row links via `related_id=orig.id`. |
| `handler-quote-api.go` | — (shared helpers) | `sideToBidAsk`, `validateAccountFormat`, `mapFixErrMsg`, `quoteOrderRecord` struct, `lookupQuoteByID` (FOR UPDATE), `lookupQuoteByHnxID` (Queue only), `markQuoteQueued`, `markQuoteRejected`. |
| `handler-quote-fix-builder.go` | — (4 builders) | `buildQuoteFIXReq`, `buildQuoteRequestFIXReq`, `buildQuoteCancelFIXReq`, `buildQuoteResponseFIXReq`. Constants: `ordTypeTTDT="S"`, `quoteRespAccept="1"`, `quoteCancelAll="4"`. |
| `handler-quote-nats.go` | — (8 consumer funcs) | NATS inbound handlers — see dispatch table. |

## NATS dispatch (`handler-api.go` ~`:6080-6124`)

| Case | tag 537 | Handler | Effect |
|---|---|---|---|
| `AI` (own echo) | — | `insertLogHnx` | senderCompID == fixSenderCompID → just log |
| `AI` | `1` | `handleAI_Place` | Place ACK → hnx_status=Queue, set order_id_hnx; no placer row → `createQuoteFromBroadcast` |
| `AI` | `2` | `handleAI_Amend` | Amend ACK → hold(new)+release(old), update root, mark edit Completed |
| `AI` | `3` | `handleAI_Cancel` | Cancel ACK → release G3, hnx_status=Canceled, is_deleted=1 |
| `AI` | `4` | `handleAI_RaceLoser` | peer matched first → hnx_status=16 Invalid_Counterpart_Matched + release |
| `AI` | `5` | `handleAI_CpChanged` | placer changed counterparty → hnx_status=17 Invalid_Counterpart_Changed + release |
| `AJ` (own echo) | — | `handleAJ_AcceptEcho` | mark acceptance row Queue, set order_id_hnx |
| `8` ExecReport | — | `handleExecReport_TTDT` / `handleExecReport_BCGD` | `lookupTxnType` → outright_ttdt: ExecType=3 → all rows Completed, ExecType=4 → Canceled |

## DB schema (migrations)

| Migration | Change |
|---|---|
| `000022_alter_table_order_transaction_type` | added `transaction_type` to `order` (nullable; legacy rows NULL = BCGD) |
| `000023_alter_table_order_ttdt_fields` | added `regist_id VARCHAR(512)` (tag 513), `is_visible INTEGER DEFAULT 0` (tag 1111) |
| `000024_alter_table_logs_transaction_type` | added `transaction_type` to `logs` so RetryUpdateHnx cron can filter |

## Constants (`datastruct/constant.go`)

- `Transaction_Type_TTDT = "outright_ttdt"` (~:88) vs `Transaction_Type_BCGD = "outright_bcgd"` (~:87).
- `HNX_Status_Queue = 2`, `HNX_Status_Invalid_Counterpart_Matched = 16` (537=4), `HNX_Status_Invalid_Counterpart_Changed = 17` (537=5).
- All TTDT rows pinned by `transaction_type='outright_ttdt'` in WHERE clauses.

## Request body structs (`datastruct/core-api.go` ~`:391-431`)

`NewQuoteBodyRequest`, `AmendQuoteBodyRequest`, `CancelQuoteBodyRequest`, `AcceptQuoteBodyRequest`, `QuoteResponse` (output: messageId, status, orderId, clOrdId).

## Column / correlation invariants

- `order_id` stores `clOrdID` (set by `markQuoteQueued`) → NATS consumer looks up rows by ClOrdID echo.
- `order_id_hnx` stores HNX SHL (set on 537=1 ACK).
- `related_id` links Amend's edit row → root row; Accept's row → orig peer quote.
- `createQuoteFromBroadcast` idempotency: SELECT existing by `order_id_hnx + transaction_type=TTDT` → skip if found.

## Phase status (2026-05)

- Phase-02: routes + handlers + INSERT + G3 hold/release + FIX builders + NATS consumers — DONE.
- Phase-03: gRPC wire to FIX gateway (`clientGrpc.SendQuote/SendQuoteRequest/SendQuoteCancel/SendQuoteResponse`, `proto/grpc_api.proto`) — gateway side TBD.
- Phase-04: NATS consumer for inbound from HNX — implemented in `handler-quote-nats.go`; verify in integration test.

FE counterpart → `terminal-fe-map.md`.