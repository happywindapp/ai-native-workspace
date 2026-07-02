# Carbon Phase-1 Test Scenarios

Source: `documents/...Test HTGD carbon.xlsx`. Test period: 08-14/04/2026 (10 days). Round 2: 05/2026.

## Test accounts (15)

| # | Account | Subject | Product | Note |
|---|---|---|---|---|
| 1-2 | `011CCB011{1,2}` | ? | TC | VSDC pre-open |
| 3-5 | `011CCB011{3,4,5}` | EMIT | HN | VSDC pre-open, 5M balance |
| 6-7 | `011CCB011{6,7}` | - | TC | TVLK self-open |
| 8-10 | `011CCB011{8,9}` + `0120` | - | HN | TVLK self-open |
| 11-12 | `011FCB012{1,2}` | Foreign | TC, HN | TVLK opens trading code (GD 2205) first, ID = MST |
| 13-14 | `011CCB012{3,4}` | Local individual | — | **Negative — expect reject** |
| 15 | `011CCB0125` | - | - | TVLK self-open |

## VSDC GD codes

| Code | Function |
|---|---|
| 2012 | Open equity (CKCS) account |
| 2205 | Issue foreign-investor trading code |
| 4251 | Register Carbon account |
| 4204 | Cancel Carbon account registration |
| 4292 | Change Carbon account type |
| 4211 | Confirm KQGD (MT598.305) |
| 4227 | Allocation notification (MT598.010) |
| 4228 | Cancel allocation (MT598.011) |
| 3331 | Carbon deposit |
| 3332 | Carbon withdraw |
| 4212/4213/4216/4217/4220/4222 | Internal MT518/598 payment messages |

## Field codes (test xlsx)

- **ID type:** 1 = CMT · 2 = TradingCode · 3 = ĐKKD · 4 = Other.
- **Org type:** 3 = Cá nhân TN · 4 = Cá nhân NN · 5 = Pháp nhân TN · 6 = Pháp nhân NN.
- **Subject type:** 1 = Cơ sở phát thải · 2 = Chủ dự án TC · 3 = Tổ chức khác.
- **Account-type registration:** 1 = Tín chỉ · 2 = Hạn ngạch.
- **Nationality:** numeric 1-251, VN = 234.

> Note: the test-xlsx "Acc type reg" code (1=Tín chỉ, 2=Hạn ngạch) matches the InfoGate `Cac-bonType` enum. Do not confuse it with the value-layer string `accountTypeName` ("quota"/"credit").

## Negative cases (must reject)

Qty not lot-aligned · qty ≤ 0 · price ≤ 0 · price outside ceiling/floor · price not tick-aligned · price below TC minimum · order outside session hours · account not registered with VSDC · wrong account type (HN account placing a TC order or vice versa).

## Trade rules baseline (Phase-1 test snapshot)

- **HN:** band ±10% (first-day ±25%), tick 1đ, lot 1, reference price = VWAP.
- **TC:** no band → ±10% from day 6, tick 1→100đ from day 7, lot 1→10 from day 8.
- Hours 09:00-11:30 / 13:00-14:45.

> Band/tick/lot changed during the test window — always load current values from Core G3.

## Reports

- **BS001** — intraday inquiry.
- **QS001** — Mẫu 10/TTCB aggregated KQGD.
- **QS004** — EOD reconciliation FileAct `.par` + `.csv`.

## Test registration

Test-registration documents are filed under `documents/01_Project Overview/Công văn/Đăng kí kiểm thử đợt 01_08042026/`. Round-01 UAT (08/04-14/04) is done; round 02 (05/2026) pending HNX confirmation.