# Carbon Business Flows (Phase 1)

The Carbon Phase-1 business flows F1-F13. For the STP pipeline mechanics see `stp-end-to-end-flow.md`; for MT message format see `financial-messaging`.

## Account flows

### F1. Register Carbon trading account
Terminal → G3SB info → MT598.301 (`23G:NEWM`, `22H::ACCT//AOPN`, `22F::TPTY//{EMIT/PROJ/ORGA}`, `22F::ACTP//{QUOT/CRDT}`) → STP Hub → VSDC → MT598.301 reply (`25D::IPRC//PACK/REJT`).

### F2. Change account type (Hạn ngạch ↔ Tín chỉ)
MT598.303 (`77E:MODE`, `22H::ACCT//MODE`) → VSDC reply MT598.116.

### F3. Cancel account registration
Balance = 0 + no outstanding settlement → MT598.301 (`22H::ACCT//ACLS`). Phase 1 deferred (D1) — `TemplateCarbonRegisterAccount` hardcodes `AOPN`.

### F4. Change investor info
MT598.303 with `95S::ALTE//VISD/TXID/VN/<MST>` inside `70E::ADTX`.

### F5/F6. Deposit / Withdraw — DCC-initiated
DCC (Bộ NN&MT) → VSDC MT540 (deposit) / MT542 (withdraw) → VSDC → TVLK MT544/546 (reject = MT548) → STP Exec → G3 `/equity/instrument/deposit|withdrawal` (transactionCode `D|W`).

> **Differs from Bond:** Bond deposit/withdraw is TVLK-initiated; Carbon is DCC-initiated. Phase 1 not wired on the MW side (D2 deferred).

## F7. Place buy/sell order (CORE FLOW)

### Pre-match
1. TMD opens Carbon Terminal → Buy/Sell tab → enters product + account + qty + price + payment method.
2. Terminal → MW → OMS queries G3: `/equity/cash/balance-g3` + `/equity/instrument/overriding-info`.
3. TMD confirms:
   - Buy: OMS → G3 `/equity/cash/hold` (amount = price × qty + fee).
   - Sell: OMS → G3 `/equity/instrument/hold` (qty).
4. TMD re-enters the same order on the **HNX Web Terminal** (Phase 1 — dual entry, no API order entry yet).

### Post-match
5. VSDC → MT518 (`22H::BUSE//BUYI|SELL`) → Terminal shows "Chờ XN TT".
6. Invalid: VSDC → MT598.000 (`23G:CANC`, `70D::REAS`) → "Bị VSD từ chối".
7. TMD clicks Confirm → MW → STP Exec sends MT598.305 (`77E:TRADE`, `25D::STAT//CONF`, `20C::PREV` = MT518 ref).

### Settle
8. Buy: OMS → VietinBank `/remittance-add` (transType=910, customer account → HSC account @ VTB), polls `/remittance-inq`.
9. VSDC → VTB MT518; VTB transfers → MT900/910 → VSDC.
10. VSDC → MT544 (buy) + MT546 (sell) → Terminal "Chờ phân bổ T0".
11. Buy: OMS G3.5 import BUY → G3.2 release cash. Sell: G3.4 release asset → G3.5 import SELL → VTB withdraws to HSC account (transType=900).
12. Terminal "Đã phân bổ T0".

### EOD
13. TMD clicks "XN kết quả phân bổ" → STP Exec MT598.010 (`77E:CASH`).

## F8. Amend order

- **Before match:** edit directly on Carbon Terminal (payment method, qty, price, account, note) + HNX Web Terminal.
- **After confirm — proprietary-error correction:** check "Sửa lỗi tự doanh" → change customer account → HSC proprietary account → MT598.305 (`25D::STAT//REJT`, `70D::REAS`) → OMS unholds customer cash/asset + holds proprietary account → awaits VSDC adjusted MT518. **Phase 1 does NOT implement this** (QC ver03 Điều 14, Phase 2).

## F9. Cancel order after match
**No direct cancel.** Not confirming settlement before the 16:00 cut-off → VSDC auto-removes + MT518 `23G:CANC` → OMS unholds back to customer account.

## F10. End-of-day reconciliation
- 16:00 — TMD checks the Terminal summary.
- 16:30 cut-off — send MT598.010 confirming allocation (correction = MT598.011 cancel-confirm, `20C::PREV` references the 010).
- 17:00 — VSDC FileAct `.par` + `.csv` KQGD → STP Exec parses → Terminal does a 3-way comparison (Carbon Terminal / G3 / VSDC), exports a 3-sheet `.xls`.

CSV format: `{mã TVLK};{Tên};{Ngày GD};{Ngày TT};{Số ĐD};{Mã HN/TC};{Số XN SGDCK};{Loại lệnh};{Trạng thái};{TK mua};{TK bán};{GT khớp};{SL khớp};{Mã TVLK mua};{Mã TVLK bán}`.

## F11. Status × G3 API mapping

| Step | HNX | VSD | G3 |
|---|---|---|---|
| 2 | Lệnh mới | - | G3.1 cash hold + G3.3 asset hold |
| 4.1 | Hoàn tất | Chờ XN TT | - |
| 5 | Hoàn tất | Đã XN TT | - |
| 6.3 | Hoàn tất | Chờ phân bổ T0 | Buy: G3.5 import → G3.2 release cash. Sell: G3.4 release asset → G3.5 import |
| 6.4 | Hoàn tất | Đã phân bổ T0 | - |

G3 API roles: G3.1 cash hold · G3.2 cash release · G3.3 asset hold · G3.4 asset release · G3.5 account contract (import trade).

**Import/release failure:** status "Phân bổ thất bại"; logged in order History; **no auto-retry** — user resolves manually.

## F12. New Carbon symbol (Phase 1 manual)
DCC → MT598.007 (HN) / MT598.008 (TC) → VSDC broadcasts to TVLK → STP Exec emails BO → BO creates the G3 instrument manually.

## F13. Freeze/unfreeze credit (Phase 2 — deferred)
Bộ NN&MT document → VSDC → MT508 to TVLK (`93A:FROM//AVAL + 93A:TOBA//PLED` = freeze; reverse = unfreeze) → TVLK freezes/unfreezes in Core G3 same day.