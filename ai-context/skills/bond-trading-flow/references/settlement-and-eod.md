# Settlement E2E + End-of-Day Reconciliation

Traces settlement across BondOMS + BondTradingMiddleware + HSC_STP + VCB. There is **no explicit EOD batch** — settlement is event-driven via polling (15s BondOMS crons + HTTP-triggered Middleware jobs). Critical for debugging "order stuck at vsdStatus=X".

## Pipeline overview

HNX FIX match → MT518 (VSD obligation) → VCB bank transfer → MT598 confirm → MT544 (buy) / MT546 (sell) → BT Phân Bổ → G3SB contract + cash/bond release → `vsdStatus=T0(6)`.

State is owned by BondOMS Postgres (`order.hnx_status`, `vsd_status`, `bank_status`). Middleware polls STP and writes back via the OMS `/amend` endpoint.

## Status machines

```
VSD: none(1) --[MT518 match]--> waiting_confirm(2)
  --[BondOMS amend vsd+bank=WaitingForConfirm]--> callTransferToVCB
  --[Middleware 598 confirm]--> callToConfirmPaymentSTP (MT598 -> VSD)
  --[VSD confirms]--> waiting_confirm_update(3) or confirmed(4)
  --[MT544 Buy / MT546 Sell]--> waiting_T0(5)
  --[BT phân bổ: staffId=vsd, makeG3Order]--> T0(6) = DONE
  Side: trade-error → denied(7); trade-cancel → cancel(8); alloc fail → AllocatedFail(9)

Bank: none(1) --[MT518 match]--> waitingForConfirm(2)
  --[callTransferToVCB OK]--> RetryBankDeposit polls
  --[VCB GetTransactionStatus success]--> success(4)
  --[fail]--> failed(5)   ; --[manual]--> confirmed(3)
```

## HNX → T0 pipeline (code-level)

1. **HNX match → `hnx_status=Completed(6)`** — FIX ExecutionReport → NATS → `handleOne` → `insertLogHnx`. `RetryUpdateHnx` (15s cron) reads log → sets `order.hnx_status=6`.
2. **MT518 match → `vsdStatus=WaitingForConfirm(2)`** — VSD sends MT518 `.fin` to STP. Middleware `runGetStatusVSDFromSTP` polls STP `GET /outputs/list?type=payment-obligation`. Match keys: `orderIdMatch` + `transaction_type` (BUYI/SELL) + `custodyId` + `bondCode` + `price` + `qty` + `amount`. OK → OMS `/amend` `vsdStatus=2, bankStatus=2`; fail → `/update-history` with a Vietnamese mismatch message.
3. **VCB transfer → `bankStatus=Success(4)`** — `AmendOrderHandler` detects `vsd=WaitingForConfirm + bank=WaitingForConfirm` → `callTransferToVCB` → `vcbInstance.Transfer()`. `RetryBankDeposit` polls `GetTransactionStatus()`. OK → `bankStatus=4`.
4. **MT598 confirm payment → VSD ack** — Middleware filters `hnxStatus=completed + vsdStatus=confirmed + orderIdMatch`, checks `checkConnect` idempotency, then `callToConfirmPaymentSTP(orderIdMatch, true)` → STP. Log key `SEND_598_STP`.
5. **Trader "Phân Bổ" → STP `/allocation/confirm`** — FE `POST /api/commands/confirm` → `runSendConfirmSTP` → STP `/allocation/confirm`; creates a `bondDeposit` record (type=confirm).
6. **MT544 (Buy) / MT546 (Sell) → `vsdStatus=WaitingT0(5)`** — Middleware polls STP `/outputs/list?type=increase-amount` (544) or `decrease-amount` (546). Match: **`linked_ref = orderIdMatch` only**. OK → `vsdStatus=5`.
7. **BT Phân Bổ (`makeG3Order`) → `vsdStatus=T0(6)` = SETTLEMENT COMPLETE** — Middleware → OMS `/amend` `staffId="vsd", vsdStatus=WaitingForAllocateT0`. `AmendOrderHandler` calls `makeG3Order`: Buy = `makeTransactionBond` → `makeCashRelease`; Sell = `makeStockRelease` → `makeTransactionBond`.

## MT message matching details

Line refs in `BondTradingMiddleware/src/services/handleResFromSTP.js`.

- **MT518 — Payment Obligation.** Filter: `hnxStatus=Completed(6)`, `vsdStatus` none/empty, `orderIdMatch != null`. STP `GET /outputs/list?type=payment-obligation`. Match: `matched_ref=orderIdMatch`; `transaction_type` computed via `getMyPerspective` (cross-firm swap applied); `party_account=custodyId`; `instrument_code=bondCode`; `deal_price=price`, `unit=qty`.
- **MT544 — Increase Amount (Buy).** Filter: `hnxStatus=6`, `vsdStatus=confirmed(4)`, `orderIdMatch != null`, `getMyPerspective(data).myBidAsk === "buy"`. Match: `linked_ref = orderIdMatch` ONLY. Success → `vsdStatus=5`, log `RECIVED_544_STP`.
- **MT546 — Decrease Amount (Sell).** Same as 544 but `myBidAsk === "sell"`. Match: `linked_ref` ONLY. Log `RECIVED_546_STP` (tracking key bugged — see `known-bug-patterns.md`).

## Business timeline (Quy trình Thanh toán §4.2 — Bước 1-20)

| Time | Actor | Event |
|---|---|---|
| T morning/afternoon | HNX | BCGD matching (09:00–11:30, 13:30–15:00) |
| T 15:30 | HNX → BT | ExecutionReport (Bước 3) |
| T 15:30 | VSD | Receives KQGD, validates (Bước 4); invalid → Bước 5-6 |
| T+0 15:30 | BT → NHTT | Submit buy-side cash payment (Bước 10) |
| T+0 15:30 | BT → VSD | MT598 accept KQGD (Bước 11) |
| T+0 15:45 | VSD → BT | MT544/546 (Bước 14) — after NHTT MT910 confirm |
| T+0 16:30 | BT | Allocate bond/cash (Bước 15/17) + MT598 result (Bước 19) |
| T+0 17:00 | VSD | Consolidated report (Bước 20) |

VSD validity checks (Bước 4-6): custody account exists, settlement date present, TVLK not suspended, no duplicate confirmation number, bond code registered, seller has bond balance, buyer is a professional investor. Invalid → cancel KQGD → HSC handles the error.

SWIFT messages: **MT518** payment obligation (VSD→HSC, after match, locks bond); **MT544** increase amount (VSD→buyer TVLK); **MT546** decrease amount (VSD→seller bank); **MT598** generic settlement (HSC↔VSD: registration/confirm/result); **MT910** payment confirm (NHTT→VSD). Wire format details belong to `financial-messaging`.

## UI status labels

The "Sửa lệnh" dialog shows 3 status fields: HNX / NHTT / VSD.

- **NHTT (bank):** Chờ xác nhận nộp tiền → Đã xác nhận nộp tiền → Nộp tiền thành công → (Nộp tiền thất bại on NAK/timeout).
- **VSD (allocation):** Chờ xác nhận thanh toán (after MT518) → Xác nhận Kết quả giao dịch (QLGD confirms MT518).

Progression after MT518: MT518 from VSD → QLGD confirms → BT sends payment message → NHTT ACK → VCB pays → NHTT→VSD MT910 → VSD issues MT544/546 → BT allocates. The newer model splits NHTT out separately to track bank payment; QLGD confirms MT518 manually before cash is submitted.

## End-of-Day reconciliation — 3-way matching

Verify each matched trade across 3 parties: Bond Terminal (HSC) — G3 (HSC) — VSD. Time: T+0 16:30–17:00 (after allocation). Owner: Bond Terminal.

- **Send KQPB** (Kết Quả Phân Bổ): UI Báo cáo → So khớp GD cuối ngày. Metrics exclude proprietary accounts. Action "Gửi điện xác nhận". **Deadline 16:30 T+0.**
- **Cancel & resend KQPB** if it diverges from VSD: re-match + detail → adjust wrong statuses → cancel the old message (X on the **most recent** successful message only) → refresh → resend.
- **3-way match:** after 17:00 VSD sends a file → BT auto-matches. Comparison table per row: order ID, status, quantity, settlement value — matched across all 3. Export Excel with 3 sheets (BT, G3, VSD).

### Discrepancy scenarios
| Scenario | Bond Terminal | G3 | VSD | Cause |
|---|---|---|---|---|
| VSD reject | Hoàn tất | Hoàn tất | **Hủy** | VSD rejected (insufficient funds, validation fail) |
| G3 slow | Hoàn tất | **Chờ xác nhận** | Hoàn tất | G3 not yet processed |
| BT slow allocate | **Chờ phân bổ** | Hoàn tất | Hoàn tất | Allocation delayed |
| VSD not received | Hoàn tất | Hoàn tất | (blank) | VSD missing info |
| Proprietary pending | **Chờ phân bổ TO** | — | — | Proprietary trade not allocated |
| Order-number mismatch | diff ID | diff ID | — | BT vs G3 ID mismatch |

Manual action on discrepancy: open detail → check the wrong field → verify BT order/allocation → verify G3 trade match → check VSD NAK reason → fix (BT re-allocate / G3 manual + IT / VSD NAK → fix + resend MT518) → cancel old message + resend KQPB.

## Debugging guide — order stuck at status

| Stuck at | Likely cause | Check |
|---|---|---|
| `vsdStatus=1` | MT518 not matched | STP outputs payment-obligation, matching fields |
| `vsdStatus=2` | bank pending or 598 not sent | `bankStatus`, `RetryBankDeposit` logs, `checkConnect` |
| `vsdStatus=3` | MT518 re-confirm pending | STP outputs, `matchedItems.length` |
| `vsdStatus=4` | MT544/546 not matched | cross-firm `getMyPerspective` bug? `flagAllocate`? STP outputs |
| `vsdStatus=5` | BT phân bổ not triggered | Middleware confirm flow, `bondDeposit` records |
| `bankStatus=2` | VCB pending | `RetryBankDeposit` logs, VCB API |
| `bankStatus=5` | VCB failed | VCB error, manual retry |

There is **no EOD reconciliation batch** — silent failures stick until manual intervention.