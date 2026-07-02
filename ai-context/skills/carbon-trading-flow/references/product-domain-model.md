# Carbon Product Domain Model

The Carbon trading domain — what is traded, how products and accounts are classified.

## Two commodities

| Element | Hạn ngạch (QUOTA) | Tín chỉ (CREDIT) |
|---|---|---|
| Product-code length (DCC convention) | 6 chars | 9 chars (3-char mechanism + 4-char project + 2-digit year) |
| Short-code examples | `VN2024`, `VN2025`, `VN2026` (VN+year) | `CHTAN3D26`, `VCSBLUE23`, `ACRBNEU24` |
| ISIN (STP 35B) | `VN000000CARB` pattern (12-char, official per VSDC v0.5) | same pattern |
| Serial length | 16 chars (e.g. `VN20280027716729`) | 19 chars (e.g. `CHTAN3D260000099951`) |
| MT598 new-listing sub-type | `007` (77E:ISIN, SPRO:ALOW) | `008` (77E:ISIN, SPRO:CRDT) |
| MT598 de-listing | — | `100` (77E:DLST) |
| `:22F::ACTP` qualifier | `QUOT` | `CRDT` |
| Reference price | VWAP of prior day | not applicable |

Two code formats co-exist: the 6/9-char DCC convention at the value layer, and 12-char ISIN `VN000000CARB` at the STP layer.

> Band / tick / lot changed daily during the Phase-1 test window — **load them from Core G3, never hardcode**. (Phase-1 test snapshot: QUOTA band ±10% / first-day ±25%; CREDIT band none → ±10% from day 6; ticks/lots scaled up day 7-8.)

## The 1-1 trading rule (canonical, FE Carbon-Terminal v2026-05)

| `accountTypeName` | Trades | Code length |
|---|---|---|
| `quota` (NĐT Hạn ngạch Carbon) | **only Hạn ngạch** | 6 chars |
| `credit` (NĐT Tín chỉ Carbon) | **only Tín chỉ** | 9 chars |

- Decided 2026-05-06; `allowanceCarbon`/`creditCarbon` renamed → `quota`/`credit` (lowercase) on 2026-05-07 across FE+BE+DB.
- Carbon-OMS table `order.product_type` = enum `('quota','credit')` default `'credit'`; API `/equity/order/new` field `productType` empty → falls back to `credit`.
- Two separate tabs in `/customers-list`, `/account-management` (NOT merged — business decision).
- VSDC tag `:22F::ACTP//QUOT|CRDT` keeps uppercase at the STP layer (separate from this value layer).

## Three investor subject types — `:22F::TPTY` in MT598.301

| Code | Type | Can trade |
|---|---|---|
| `EMIT` | Cơ sở phát thải (emitting facility) | Hạn ngạch **OR** Tín chỉ — pick one |
| `PROJ` | Chủ dự án tín chỉ (credit-project owner) | Tín chỉ only |
| `ORGA` | Tổ chức khác (other org) | Tín chỉ only |

`investor_type` integer `1|2|3` maps to `EMIT|PROJ|ORGA`. **Hard FE rule:** `InvestorType.tsx` forces `investor_type=1` when account type = quota → QUOT always travels with EMIT.

## Account classification — `TYPE//` field inside `70E::ADTX`

| Code | Meaning |
|---|---|
| `DOMIND` | Domestic individual |
| `FORIND` | Foreign individual |
| `DOMCORP` | Domestic organization |
| `FORCORP` | Foreign organization |
| `GOVT` | State legal entity |

Inference from ID type + nationality: VN+IDNO/CCPT→DOMIND · FOREIGN+CCPT/ARNU→FORIND · VN+CORP/OTHR→DOMCORP · FOREIGN+FIIN/OTHR→FORCORP.

## ID types — `95S::ALTE//VISD/<type>/VN/<id>`

| Code | Meaning |
|---|---|
| `IDNO` | CMND/CCCD (national ID) |
| `CCPT` | Passport |
| `CORP` | ĐKKD (business registration) |
| `OTHR` | Other certificate |
| `FIIN` | Trading Code for foreign organization |
| `ARNU` | Trading Code for foreign individual |
| `GOVT` | Government agency |
| `TXID` | Tax code (inside 70E::ADTX) |

> For the full FE↔STP↔MT doctype mapping chain, see `account-rules.md`.

## Account number format

Carbon trading account (derived from the equity CKCS account):
- Pattern: `{3-char custody-member code}` + `{1 char ∈ C|P|F|E|A|B}` + `{≤6 digits}` → max 10 chars.
- `011` = HSC custody-member prefix; e.g. `011CCB0111`, `011FCB0121`.
- 4th char: `C` individual · `F` foreign · `P` proprietary (inferred) · `A/B/E` TBU (HNX rule, semantic unconfirmed).
- The first 3 chars must not collide with another custody member's code.

## Trading sessions

- Morning: 09:00–11:30 · Break: 11:30–13:00 · Afternoon: 13:00–14:45 (negotiated trades only).

### InfoGate status codes (FIX tags)

| Tag | Field | Codes |
|---|---|---|
| 340 | TradSesStatus | 1 receiving orders · 2 paused · 13 receiving ended for today · 90 awaiting orders · 97 market closed |
| 326 | TradingStatus | 0 stopped · 1 normal · 2 special |
| 327 | ListingStatus | 0 listed · 1 not listed · 2 newly listed · 3 de-listed |

## Two trade forms

1. **Thỏa thuận điện tử (TTĐT)** — negotiated electronic matching; addressed to 1 member / a group / Public; optional Anonymous.
2. **Báo cáo giao dịch (BCGD)** — two parties agreed off-system, reported into HNX. No anonymous; must pick counterpart member + counterpart account.

## Business cut-off timings (QC Carbon ver03)

- **15:30** — TVLK sends MT598 accept/reject KQGD (after receiving MT518); submits payment-removal request (Mẫu 11/TTCB) if needed.
- **15:45** — VSDC processes trade errors, sends adjusted MT518.
- **16:00** — VietinBank transfers funds (TVLK deposits ~09:00-16:00).
- **16:30** — TVLK sends MT598.010 notifying cash+carbon allocation result to investor (final cut-off).
- **17:00** — VSDC sends FileAct summary KQGD report (`.par` + `.csv`); TVLK reconciles.