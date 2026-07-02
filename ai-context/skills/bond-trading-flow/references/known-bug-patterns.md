# Known Bond Bug Patterns

Recurring bug classes in the Bond/TPRL system. Each entry: symptom → cause → fix signature. Use this to recognize a bug class fast, then hand the actual fix to the `fix`/`debug` skill.

## Orientation bugs (the #1 recurring class)

The root cause family: a cross-firm row stored in Orient 2 (HSC at reciprocal) needs `bid_ask` flipped and the G3 op routed to the HSC account. Forgetting the flip or mixing a swapped account with a raw `bid_ask` flips cash↔bond.

### Pattern A — G3 account + bidAsk not paired (cross-firm wrong asset)
- **Symptom:** cross-firm amend — HSC buyer triggers a Sell-path G3 op (releases stock instead of cash), G3SB rejects, or wrong contract direction.
- **Cause:** code passes a swapped `orderAccountCallG3` together with a raw `DataUpdate.BidAsk`. Account and direction are out of sync.
- **Fix signature:** always use `(orderAccountCallG3, orderTypeCallG3)` as a pair. (Fixed in the VSD `staffId="vsd"` amend block and `makeG3Order` — both now use `orderTypeCallG3`.)
- **Audit rule:** grep for any G3 call mixing `orderAccountCallG3` with `DataUpdate.BidAsk`.

### Pattern B — payment-confirm missing the Orient-2 HSC-sells case
- **Symptom:** cross-firm Orient 2 (HSC at reciprocal sells) — `vsdStatus` stuck at `OrderResultConfirm(10)`, UI hangs at "Xác nhận KQGD".
- **Cause:** `AmendOrderHandler` had two separate `if` blocks that did not cover HSC-at-reciprocal.
- **Fix signature:** a single merged predicate `hscAtOrderSide || hscAtReciprocalSide` combined with `payload.VSDStatus == OrderResultConfirm` → set `VSD_Status_PaymentConfirmed`.

### Pattern C — `createOrderOutrightFromHnx` swap by wrong test
- **Symptom:** cross-firm CP-accept order — invisible to the HSC trader on the FE; 518/544/546 matching skips; hold/release uses the counterparty's account.
- **Cause:** swap decided by `ReciprocalAccount != ""` instead of checking which side is `"011"`. Row stored `order_company=CP`, `bid_ask` from CP perspective.
- **Fix signature ("Fix B"):** `getMyPerspective()` in Middleware — interprets the row from HSC's perspective at *read* time (does not mutate DB). Applied to 518/544/546 matching. The FE flips perspective in `displayRow()`. A full BondOMS-side root-cause fix is deferred.
- **Detect:** `SELECT * FROM "order" WHERE order_company != '011' AND reciprocal_company = '011' AND is_deleted = 0;`

## Amend reconcile bugs

### Cross-firm BOND amend ACK — lookup fails (FIXED)
- **Symptom:** HSC self-initiated cross-firm amend reducing qty/price — HNX confirms but G3 does not `release(old)+hold(new)`. Log: `approve cross-firm BOND lookup edit fail: sql: no rows in result set`.
- **Cause:** the SELECT in the BOND amend-ACK block omitted `order_id_root, order_company, reciprocal_account` → `DataUpdate.OrderIdRoot=""` → downstream edit-row lookup fails.
- **Fix:** SELECT now includes those fields + filters `order_type=Order_Root` (avoids picking arbitrarily across root+edit rows both at `hnx_status='2'`). Single-amend only.

### `RetryUpdateHnx` case "9" — partner-initiated reject not reverted (FIXED)
- **Symptom:** HNX rejects a partner-initiated cross-firm amend → G3 does not revert.
- **Cause:** case "9" had no Orient-2 branch; main/recip lookups used a fragile `order_type='order_root'` filter.
- **Fix (3 parts):** (1) Orient-2 branch — flip `bid_ask`, look up root by `id<>$2`, `hold(old)` then `release(new)` on ReciprocalAccount, `break`. (2) main/recip lookups filter `hnx_status=Completed(6)` to anchor the active prior row. (3) ordering swap — `hold(old)` first, `release(new)` after (release fail → over-hold, safe).

### Partner-initiated amend double-hold (FIXED)
- **Symptom:** partner-initiated cross-firm amend post-match → double-hold.
- **Cause:** the post-match branch of `updateOrderFromPartnerHnx` only held the new qty, never released the old.
- **Fix:** SELECT adds `price, quantity`; post-match applies the "hold max(old,new)" rule — `shouldSwitch` → `release+hold` atomic; else no-op.

## Settlement / MT matching bugs

### MT544/546 matching too loose
- **Symptom:** an MT544/546 matches the wrong order, or fails to match a cross-firm one.
- **Cause:** matching uses `linked_ref = orderIdMatch` ONLY — no price/qty/account/bondCode check.
- **Fix direction:** tighten the match keys (deferred).

### External-trade 1-row rule — wrong MT type tested
- **Symptom:** simulating an MT544 for an external trade never matches any row.
- **Cause (intentional rule, not a bug):** an external trade (HSC vs another member) creates **1 row only**, in the partner's perspective. If that row is `bid_ask=sell`, only an MT546 will match it.
- **Apply:** before simulating an MT, check internal vs external (`order_company` vs `reciprocal_company`). External → test only the MT type matching the single row's `bid_ask`.

### MT546 tracking-key copy-paste
- **Symptom:** MT546 log entries tagged with the MT544 key.
- **Cause:** `errorMessage.js` has `RECIVED_546_STP: "Recived_544_STP"` — copy-paste typo. Still present.

## BondOMS lifecycle holes

| ID | Pattern | Symptom | Status |
|---|---|---|---|
| NewOrder asset leak | `success=true` set *before* `SendNewOrderCross` gRPC; fail path updates status but never releases | gRPC/FIX fail → cash/bond held with no order | open |
| Cancel double-spend | Cancel releases assets → commit → `SendCrossOrderCancel` gRPC; HNX may then reject (39=9/10) | asset released client-side while HNX keeps the order | open |
| Cancel state machine dead | states 10/11/13/14 never set; `CancelOrderHandler` only does gRPC + soft-delete | controlled-cancel flows untracked | open (greenfield) |
| OrdStatus=11 mishandled | spec 3.3.12 auto-cancel `(35=8,150=4,39=11)` treated as generic ExecType=4 | counterparty-changed auto-cancel silently mishandled | open |
| Amend cancel branch — no G3 release | `RetryUpdateHnx` cancel branches update DB status only | HNX auto-cancel leaves G3SB hold | TODO |
| `staffId=vsd` no auth | `AmendOrderHandler` trusts client `payload.HNXStatus`; no auth on the route | client can drive `hnx_status` — #1 security hole | open |

## Infrastructure / cross-module risks

- **`vsd_status` has two writers** — BondOMS and Middleware both write it → race. Per spec BondOMS should be sole owner.
- **NATS rate-limited ~1 msg/s** — global mutex + `time.Sleep(1s)` in `handleOneSerial`. HNX bursts dozens/sec at match. May be masking a race in `insertLogHnx` — investigate before removing.
- **JetStream published, consumed as core-NATS** — `js.Publish` + FileStorage but `nc.ChanSubscribe` plain (not durable). Messages during BondOMS downtime are silently lost; `DLQ.{subject}` has no subscriber. Fix: durable `js.PullSubscribe`.
- **`defer rows.Close()` nil-panic** — `rows, err := Query(...); defer rows.Close(); if err != nil { return }` panics on nil `rows`. ~17 sites remain. Triggered by PG pool exhaustion from Middleware polling. Fix: move `defer` after the err check.
- **BondFIXOrderGW inbound `OnFIX44*` stubs are dead code** — the real pipeline is `FromApp → sendMsgToNats`. Do not "fix" the stubs.
- **No EOD reconciliation batch** — silent settlement failures stick until manual intervention.

## SELECT/Scan field audit (pre-commit checklist)

Many of the bugs above share a root cause: a SELECT omits a field that downstream code reads from `DataUpdate.X`, or a `Scan` hits a NULL column. Before committing any change touching an amend/settlement block:
- Verify every `DataUpdate.X` field read has a matching column in the SELECT.
- Use `COALESCE(col, '')` for nullable string columns scanned into non-pointer Go strings (e.g. `edited_id`).
- Add `ORDER BY` + `LIMIT 1` + an `order_type` filter when a query can return multiple root+edit rows.