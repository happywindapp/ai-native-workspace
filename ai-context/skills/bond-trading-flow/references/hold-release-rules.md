# Hold / Release Rules — Cash & Bond

BondOMS holds/releases cash (Buy) and bond inventory (Sell) via G3SB SOAP. This is the real-money path — a mistake means the customer loses use of cash/bonds, or carries a phantom debt.

## Call chain — BondOMS → G3SB

All operations go through G3SB SOAP `messageTransfer`.

| Action | G3SB Request | Go func | Where |
|---|---|---|---|
| Hold cash (Buy) | `CreateCashHold` HOLDTYPE=D | `makeCashHold` | `handler-core-api.go` |
| Hold bond (Sell) | `CreateInstrumentHold` HOLDTYPE=T MARKETID=TPRL | `makeStockHold` | `handler-core-api.go` |
| Release cash | `CreateCashRelease` | `makeCashRelease` | `handler-core-api.go` |
| Release bond | `CreateInstrumentRelease` | `makeStockRelease` | `handler-core-api.go` |
| Match → contract | (composite) | `makeTransactionBond` | `handler-core-api.go` |

Response XML → `RESULT_HOLD_XML` / `RESULT_CASH_RELEASE_XML` / `RESULT_ACCOUNT_CONTRACT`.

## Wrapper layer (`handler-api.go`)

- `holdAssetOrder` / `releaseAssetOrder` — switch on `bidAsk`: Buy → cash, Sell → bond quantity only.
- `checkAccountOrder` — EQD pre-check against G3SB; skipped when `orderKind == "tprl"`.
- `getTotalCashHold` — `price*qty*(1+fee)`. **Does NOT add tax.**
- `makeG3Order` — runs at match or VSD allocate. Buy = `makeTransactionBond` → `makeCashRelease`; Sell = `makeStockRelease` → `makeTransactionBond`.
- Amend hold/release — inline at the 4 submit sites in `AmendOrderHandler` (see `amend-flow.md`).

## Ordering rule

On reconcile/reject paths, **hold(new/old) before release(old/new)**. A release that fails after a successful hold leaves an *over*-hold (safe). The reverse order risks an under-hold = asset leak. (Reject case "9" fix applies this: hold(old) first, release(new) after.)

Submit sites use release-then-hold but return an error immediately on hold failure (user retries) — no best-effort rollback there.

## Cash check has 2 layers

| Layer | Where | BondPlus | TPRL/Home |
|---|---|---|---|
| EQD pre-check | `checkAccountOrder` (fail-fast UX) | ✅ runs | ❌ skipped (`orderKind=="tprl"`) |
| G3SB `CreateCashHold` | `holdAssetOrder` → `makeCashHold` | ✅ runs | ✅ runs |

The G3SB `CreateCashHold` is the **authoritative cash gate** — the G3SB ledger rejects if the account lacks funds. It runs *before* the FIX message is sent to HNX, for every `orderKind`. The EQD pre-check is only a fail-fast UX layer for retail BondPlus (the EQD view may not cover proprietary/counterparty-mirror accounts → false reject for TPRL).

## Trigger sites in the order lifecycle

| Lifecycle stage | Hold/release behavior |
|---|---|
| **NewOrder** | `holdAssetOrder` for client + same-firm reciprocal; `defer` rollback releaseList. Fail path updates status but does NOT release — see leak bug in `known-bug-patterns.md` |
| **CancelOrder** | soft-delete → release OrderAccount + same-firm reciprocal immediately → commit → send `CrossOrderCancel` gRPC |
| **AmendOrder** | trust-the-client release when `payload.HNXStatus ∈ {Canceled, Rejected}`; the 4 submit sites apply "hold vừa đủ cover" |
| **Match** | `makeG3Order` creates the contract + releases the matched portion |
| **Accept post-match** | `release(root)+hold(new)` on ReciprocalAccount, guarded `IsAccept && OrderIdMatch != nil` |

## Module boundaries

- Hold/release runs **only inside the BondOMS process** — not over NATS/Middleware.
- After match, settlement continues via HSC_STP (SWIFT `.fin`) → `vsd_status` changes but does NOT trigger more hold/release (release already done at match).
- Middleware updates `vsd_status` after polling STP — this can race with the release already done at match (`vsd_status` has two writers — see `known-bug-patterns.md`).

## Lock-user-action UI rules

Bond Terminal's "Sửa lệnh"/"Hủy lệnh" buttons are enabled/disabled by lifecycle state. General logic: intermediate states lock both the root and the edit/cancel order; the root reopens only when the flow is rejected.

### Amend — "Chờ khớp" (all members)
Editable fields: price, quantity, account, payment method.
| Step | Root order | Edit order |
|---|---|---|
| User submit OK | `Chờ thực hiện` 🔒 | `Chờ kiểm soát sửa` 🔒 |
| HNX accept | `Không hiệu lực` 🔒 | `Chờ thực hiện` ✅ (can amend/cancel) |
| HNX reject | `Hoàn tất` ✅ (can amend) | `HNX từ chối sửa` 🔒 |

### Amend — "Đã khớp TT cuối ngày" — cross-firm
Editable: price, quantity, account (no payment method).
| Step | Root order | Edit order |
|---|---|---|
| User submit OK | `Hoàn tất` 🔒 | `Chờ xác nhận sửa` 🔒 |
| Counterparty accept | `Hoàn tất` 🔒 | `Chờ kiểm soát sửa` 🔒 |
| Counterparty reject | `Hoàn tất` ✅ | `Đối ứng từ chối sửa` 🔒 |
| HNX accept | `Không hiệu lực` 🔒 | `Hoàn tất` ✅ |
| HNX reject | `Hoàn tất` ✅ | `HNX từ chối sửa` 🔒 |

### Amend — "Đã khớp TT cuối ngày" — same-firm
Same as cross-firm but skips the counterparty-confirm step.

### Amend — "Đã khớp TT ngay" (all)
FE blocks the popup — no amend allowed.

### Cancel — "Chờ khớp" (all)
| Step | Root order | Cancel order |
|---|---|---|
| User submit OK | `Chờ thực hiện` 🔒 | `Chờ kiểm soát hủy` 🔒 |
| HNX accept | `Hủy` 🔒 | `Hủy` 🔒 |
| HNX reject | `Chờ thực hiện` ✅ | `HNX từ chối hủy` 🔒 |

### Cancel — "Đã khớp" (all)
FE blocks the popup — "Lệnh đã khớp không được phép hủy".

### General rules
1. Intermediate state → lock both orders.
2. Unlock the root only on a reject (counterparty/HNX) → root returns to `Hoàn tất` or `Chờ thực hiện`.
3. An edit/cancel order at a terminal state is always locked; the one exception is `Chờ khớp` + HNX accept → the edit order becomes `Chờ thực hiện` and lives on.
4. `Đã khớp TT ngay` and `Đã khớp` cancel are blocked at the FE popup — never reaches the backend.
5. "Edit order pushed from another member (already-matched order): hold cash/bond once the amend is confirmed" — links to the partner-amend path.