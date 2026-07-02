# Bond Amend Testing Playbook

How to trace amend hold/release when a tester provides `logs` + `order` table JSON snippets + a scenario + the observed bug. Goal: trace fast and consistently, with a predictable output format.

## `logs` column reference

| Column | Meaning |
|---|---|
| `id` | sequence ID (ordering) |
| `log_type` | `create` / `success` / `fail` — outer switch in `RetryUpdateHnx` |
| `detail` | `create` / `success` / `update` / `update_same` / `match` / `cancel` — inner routing |
| `status` | 2=Queue, 5=Pending_Edit, 6=Completed, 9=Declined_Edit |
| `order_id_hnx` | ClOrdID from FIX (BOND code OR UUID) — critical for branch routing |
| `order_id` | HNX OrderID (`MRER...`) — DB lookup key |
| `orig_clord_id` | OrigClOrdID — CrossID derivation |
| `des` JSON | full FIX msg: MsgType, ExecType, OrdStatus, OrderQty, Price2, PartyID, CoPartyID, OrgCrossID, Account, CoAccount, Side |

### FIX event identification (parse `des` JSON)

| MsgType | detail | ExecType | OrdStatus | Meaning |
|---|---|---|---|---|
| `s` | `create` | — | — | outgoing new order |
| `s` | `success` | — | `2` | HNX ACK new order (Queue) |
| `8` | `success` | `3` | `2` | match confirmation (Filled) |
| `8` | `success` | `5` | `A` | amend pending (BOND ClOrdID) |
| `8` | `success` | `5` | `3` | amend approved (UUID) — spec 3.3.15 |
| `8` | `success` | `5` | `9` | amend rejected (UUID) |
| `t` | `update` | — | — | cross-firm amend ACK (BOND) |
| `t` | `update_same` | — | — | same-firm amend ACK |

## `order` column reference

| Column | Meaning |
|---|---|
| `id` | DB PK |
| `order_id_root` | original HNX OrderID — groups root + edits |
| `order_id` | current HNX OrderID (changes per amend) |
| `order_id_hnx` | BOND ClOrdID BondOMS sent (NOT an HNX OrderID) |
| `related_id` | same-firm pair grouping |
| `edited_id` | edit rows: points to root's `related_id` |
| `order_type` | `order_root` / `order_edit` |
| `order_account`, `bid_ask` | from the **initiator's** POV |
| `order_company` / `reciprocal_company` | same-firm vs cross-firm |
| `price`, `quantity` | values at the row's snapshot |
| `fee` | `CommissionBondBuy + CommissionBond` (e.g. 0.001) |
| `hnx_status` | 2 Queue · 5 Pending_Edit · 6 Completed · 9 Declined_Edit · 15 Invalid |
| `vsd_status` | 1 None · 5 WaitingForAllocateT0 |
| `history` JSON | per-amend change log |

## Key formulas

- **Hold value** — Buy cash: `price × qty × (1 + fee)` via `getTotalCashHold` (tax NOT included). Sell bond: `quantity` only.
- **`shouldSwitch`** — Buy: `getTotalCashHold(newP,newQ,fee) > getTotalCashHold(oldP,oldQ,fee)`. Sell: `newQty > oldQty` (ignore price). TRUE → `release(old)+hold(new)`; FALSE → no-op.
- **Cross-firm bidAsk swap** — `bid_ask` is from the initiator's POV. If cross-firm (`order_company != "011"`), HSC is the reciprocal → flip `bid_ask` for HSC-side G3 ops.

## Tracing procedure

### Step 1 — identify the scenario
1. Count `order_type=order_root` vs `order_edit` rows.
2. Same-firm or cross-firm? (`order_company == reciprocal_company`?)
3. HSC direction (flip `bid_ask` if cross-firm).
4. `history` JSON on edit rows — what changed (qty? price? both?)
5. Final `hnx_status` → which code path ran.

### Step 2 — chronological trace (per log row)

| FIX event | Code path | G3SB op |
|---|---|---|
| Create outgoing | `NewOrderHandler` | `holdAssetOrder` (client + same-firm reciprocal) |
| HNX ack create | `case Success detail=success` | none — DB only |
| Match confirm | `case Success detail=success` status=Completed | none — sets `order_id_match` |
| Amend submit | `AmendOrderHandler` | Sites 1-4 |
| Amend pending ACK (BOND) | BOND branch OrdStatus=A | none — Pending_Edit |
| Amend approved (UUID) | legacy case "3" | `!switched → release(A)+hold(B)`, else no-op |
| Amend rejected (UUID) | case "9" | `switched → release(new)+hold(old)` |
| Counterpart declined | case "4" | `switched` + cross-firm swap |
| FIX reject | case Fail | `switched` + cross-firm swap |
| Cross-firm amend ACK (BOND) | BOND amend-ACK block | `!switched → release(A)+hold(B)` + swap |
| HNX auto-cancel | cancel branch | TODO — DB only, no G3SB release |
| VSD allocate | `staffId=vsd` block | `makeG3Order` |

### Step 3 — compute G3SB deltas
`makeCashHold(+amt)`, `makeCashRelease(-amt)`, `makeStockHold(+qty)`, `makeStockRelease(-qty)`.

### Step 4 — output format
- **Trace table:** columns `Action | G3SB op | G3SB state`. E.g. Create → `makeCashHold(+X)` → X; HNX ack → — → X; Amend Site N → `release(-old)+hold(+new)` → new.
- **"Verify with DB" table:** columns `Row | Current hnx_status | Expected | Set by Block`.

## Vietnamese phrasing map (tester wording)

| Phrase | Meaning |
|---|---|
| "đang bị bug" | currently has a bug |
| "release bị double" | over-release (extra call) |
| "hold thêm X TP" | additional stock hold of X bonds |
| "release thêm X TP" | additional stock release |
| "lúc đầu có bút toán X" | initially has accounting entry X |
| "sau khi BT cập nhật thành công" | after Middleware processed (`RetryUpdateHnx` cron ran) |
| "BT phân bổ" | VSD allocation via Middleware (`staffId=vsd` path) |
| "BT phân bổ xong" | after `makeG3Order` executed |
| "bút toán contract" | G3SB contract tx entry (from `makeTransactionBond`) |
| "khác TVLK" / "khác thành viên" | cross-firm (different CTCK) |
| "cùng TVLK" / "cùng thành viên" | same-firm (both at HSC) |
| "sửa KL X lên Y" | amend quantity X→Y |
| "sửa giá" | amend price |

## Shortcut

If a scenario matches a previously traced one, reference it and only compute the new numbers. Do not re-derive the full flow each time.