# Bond Amend Flow

Comprehensive amend reference. Applies to any work in `BondOMS/handler-api.go` amend/cancel code. Multi-amend is **disabled in production** — single-amend only (see Multi-Amend Backlog below).

## The "hold vừa đủ cover" rule

Before this rule, amend double-held (root hold + edit hold). The rule: hold only `max(old, new)` at submit time, then settle to the active value at approve/reject.

```
shouldSwitch:
  Buy:  getTotalCashHold(newP,newQ,fee) > getTotalCashHold(oldP,oldQ,fee)
  Sell: newQty > oldQty   (price ignored for stock hold)

Submit:    shouldSwitch → release(old) + hold(new)   ; else no-op (keep root hold)
Approve:   !switched (B<=A) → release(old A) + hold(new B)  ; switched → no-op
Reject:    switched  (B>A)  → release(new B) + hold(old A)  ; !switched → no-op
Counterpart declined (OrdStatus=4): reject semantic + cross-firm swap
FIX-level reject (case Fail):       reject semantic + cross-firm swap
```

| Scenario | Submit | Approve | Reject |
|---|---|---|---|
| B ≤ A (no-switch) | no-op | release(A)+hold(B) | no-op |
| B > A (switched) | release(A)+hold(B) | no-op | release(B)+hold(A) |

Compare metric: **Buy** = `getTotalCashHold(price,qty,fee) = price*qty*(1+fee)` (tax NOT included). **Sell** = `quantity` only.

## The 3 amend paths

Routing depends on the ClOrdID format and who initiated.

| ClOrdID | Initiator | Path |
|---|---|---|
| `"BOND02025-..."` (HSC's format) | HSC | **Path 1** `AmendOrderHandler` submit + **Path 2** `RetryUpdateHnx` approve/reject |
| UUID / partner format (no "BOND") | partner, forwarded by HNX | **Path 3** `updateOrderFromPartnerHnx` |
| HSC's own echo from HNX | HSC | Path 2 `RetryUpdateHnx` case "3" |

**Critical:** partner-initiated cross-firm amend bypasses both `AmendOrderHandler` and `RetryUpdateHnx` case "3". When debugging a partner double-hold, check `updateOrderFromPartnerHnx` first — do not assume the other two are complete coverage.

## Path 1 — submit sites (`AmendOrderHandler`)

Four inline submit sites apply the rule (line numbers drift; grep `shouldSwitch`):
- Site 1 — main edit row submit
- Site 2 — reciprocal same-firm edit row submit
- Site 3 — main row submit (sell-as-main)
- Site 4 — reciprocal submit (sell-as-main)

Submit-site pattern:
```go
shouldSwitch := false
if bidAsk == datastruct.Bid_Ask_Buy {
    shouldSwitch = getTotalCashHold(newP, newQ, fee) > getTotalCashHold(oldP, oldQ, fee)
} else {
    shouldSwitch = newQ > oldQ
}
if shouldSwitch {
    var clientReleaseInfo = releaseAssetOrder(acc, bidAsk, bond, oldP, oldQ, fee)
    if !clientReleaseInfo.Success { return fiber.NewError(...) }
    var clientHoldInfo = holdAssetOrder(acc, bidAsk, bond, newP, newQ, fee)
    if !clientHoldInfo.Success { return fiber.NewError(...) }
}
```

## Path 2 — approve/reject reconcile (`RetryUpdateHnx` legacy switch)

The legacy switch handles UUID-ClOrdID events (non-BOND branch). It does the real approve/reject work — do not move its logic into the BOND branch (both fire for non-BOND events → double-execute).

| Case | OrdStatus | Meaning | Sets |
|---|---|---|---|
| "A" | A | pending ACK | Pending_Edit (5) |
| "3" | 3 | HNX approved | Completed (6); `!switched → release(A)+hold(B)` main + recip; cross-firm Orient-1 wrap |
| "9" | 9 | HNX rejected | Declined_Edit (9); `switched → release(new)+hold(old)` main + recip |
| "4" | 4 | counterparty declined | Counterpart_Declined_Edit (8); cross-firm swap + switched check |

The BOND branch also handles cross-firm HSC-initiated amend ACK (detail=update, BOND ClOrdID): `!switched → release(A)+hold(B)`.

## Path 3 — partner-initiated (`updateOrderFromPartnerHnx`)

Gate: `handleOne` case "t", `senderCompID=="HNX"`, `PartyID != CoPartyID && !Contains(ClOrdID,"BOND")` → TRUE → `updateOrderFromPartnerHnx`.

SELECT root row `WHERE order_id_root = OrgCrossID AND is_deleted = 0`. Branch by `dataRelease.OrderIdMatch`:
- `nil` → **pre-match branch**: partner amends before root matched. Release the Queue row, set all rows `hnx_status=Invalid`, insert new Queue row + hold.
- `!= nil` → **post-match branch**: partner amends after root matched. Apply post-match "hold max(old,new)" rule on the HSC account.

**Design constraint:** G3 hold/release runs *immediately on receiving FIX `t`*, not deferred to operator-confirm/HNX-approve. During the pending window G3 must hold `max(old,new)` to cover both outcomes (approve→new, reject→old). This is intentional.

## Cross-firm vs same-firm

- **Same-firm** (`order_company == reciprocal_company == "011"`): BondOMS creates **2 rows** (sell + buy); each row matches its own MT546/MT544. Skips the counterparty-confirm step.
- **Cross-firm** (HSC vs different member): BondOMS creates **1 row only**, in the partner's perspective (`order_company` = partner, `reciprocal_company = "011"`). HSC is the reciprocal. This is an intentional business rule, not a bug.

For cross-firm, determine orientation (Orient 1/2 — see `order-lifecycle.md`) and flip `bid_ask` when calling G3 on the HSC side.

## Lock-user-action interaction

Intermediate states (`Chờ kiểm soát sửa/hủy`, `Chờ xác nhận sửa`) lock both the root and the edit/cancel order. The root is unlocked only when the flow is rejected. See `hold-release-rules.md` for the full lock table.

## Multi-Amend Backlog (deferred)

Production allows single amend only. Before enabling multi-amend, ALL of these MUST be fixed:

| # | Sev | Symptom | Fix direction |
|---|---|---|---|
| 1 | HIGH | BOND amend-ACK lookup filter too strict — after amend 1, root invalidated (`hnx_status=15`); filter `order_type='order_root' AND hnx_status='2'` returns 0 rows → release/hold skipped → over-hold | Anchor active prior row: `order_id_root=CrossID, hnx_status='2'`, exclude row just ACKed, `ORDER BY id DESC LIMIT 1` |
| 2 | HIGH | `updateOrderFromPartnerHnx` entry query non-deterministic — no filter/limit/order; single `Next()` may pick edit row → wrong pre/post-match branch → invalidate matched root | Add `order_type=Order_Root`, `LIMIT 1 ORDER BY id ASC` |
| 3 | MED | pre-match release lookup missing `order_type` filter → picks edit row's price/qty → wrong release amount | Add `order_type=Order_Root` to WHERE |
| 4 | MED | `RetryUpdateHnx` BOND base-order lookup uses `id DESC LIMIT 1` — multi-amend edge cases unverified | Re-design: anchor active prior row, exclude `order_id_hnx = s.OrderIdHnx` |
| 5 | LOW | silent `log.Printf` on ~8 fail paths with release/hold side-effect — no alert | Convert to `logs.Errorf` with `MANUAL INTERVENTION` prefix |

Pre-implementation test matrix must cover: same-firm multi-amend; cross-firm HSC-init pre/post-match; cross-firm partner-init pre/post-match; reject-after-accept revert. Verify G3 sync, `hnx_status` transitions, and `edited_id` audit trail link after each amend.