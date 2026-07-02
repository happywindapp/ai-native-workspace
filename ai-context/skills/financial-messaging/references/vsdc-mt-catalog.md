# VSDC MT Message Catalog

MT types and MT598 sub-codes used in HSC carbon STP. Direction: TVLK = HSC member, VSDC = depository (BIC `VSDSVN03`), NHTT = settlement bank, DCC = climate authority (Bộ NN&MT).

## MT top-level types

| MT | Direction | Purpose |
|---|---|---|
| MT508 | VSDC→TVLK | Lock/unlock carbon credits (`:93A::FROM//AVAL` + `TOBA//PLED` = lock; reverse = unlock) |
| MT518 | VSDC→TVLK/NHTT | Trade result + settlement obligation; `:23G:NEWM` normal, `:23G:CANC` cancel/remove |
| MT540 | DCC→VSDC | Deposit (ký gửi) request — carbon initiated by DCC, not TVLK |
| MT542 | DCC→VSDC | Withdrawal (rút lưu ký) request |
| MT544 | VSDC→TVLK/DCC | Confirm debit/credit increase (deposit success, buy-side settlement credit) |
| MT546 | VSDC→TVLK/DCC | Confirm decrease (withdrawal success, sell-side settlement debit, code delist) |
| MT548 | VSDC→TVLK/DCC | Reject custody/withdrawal/settlement instruction |
| MT598 | 2-way | Proprietary message — 3-digit sub-code in tag `:12:` (see below) |
| MT900 | NHTT→VSDC | Payment confirmation (buyer bank) |
| MT910 | NHTT→VSDC | Payment confirmation (seller bank — variant of MT900) |

`13A::LINK//<MT>` ties a confirmation back to its parent (MT544→518, MT546→542).

## MT598 sub-codes (tag `:12:`, mode in `:77E:`)

| Sub | 77E mode | Direction | Purpose | Key tags |
|---|---|---|---|---|
| 001 | NORMAL | TVLK→VSDC | Open/close securities account (legacy TPRL, reused) | `:22H::ACCT//AOPN`/`ACLS` |
| 002 | NORMAL | VSDC→TVLK | Confirm account open/close | `:25D::IPRC//PACK`/`REJT` |
| 003 | — | TVLK→VSDC | Query report (FileAct pull) | `:13B::STAT//<code>` |
| 005 | BALANCE | VSDC→TVLK | EOD balance confirm (FileAct push) | `BRID 0008`=bond, `0009`=carbon |
| 006 | BALANCE | TVLK→VSDC | Reject EOD balance | `:20C::PREV//` ref 005 |
| 007 | ISIN | DCC→VSDC→broadcast | New **quota** code listed (6-char) | `:70E::SPRO//ALOW/…` |
| 008 | ISIN | DCC→VSDC→broadcast | New **credit** code listed (9-char) | `:70E::SPRO//CRDT/…` |
| 010 | CASH | TVLK→VSDC | Notify cash/carbon allocation to investor | `:70E::SPRO//` multiline |
| 011 | CASH | TVLK→VSDC | Revoke allocation | `:25D::IPRC//REJT` |
| 100 | DLST | VSDC→TVLK | Credit code delisted (MT546 follows) | `:35B::ISIN//<code>` |
| 112 | NORMAL | TVLK→VSDC | Transfer to adjust org type, no account close | `:22H::ACCT//TWAC` |
| 116 | MODE | VSDC→TVLK | Confirm account-type adjustment (response to 303) | `:25D::IPRC//ACPT`/`REJT` |
| 203 | — | TVLK→VSDC | **Legacy** type-adjustment request — replaced by 303 |
| 301 | NORMAL | TVLK→VSDC + reply | Register/unregister **carbon trading account** | `:22H::ACCT//AOPN`/`ACLS`, `:22F::TPTY`, `:22F::ACTP` |
| 303 | MODE | TVLK→VSDC | Request account-type change (quota↔credit) / investor update | `:22H::ACCT//MODE`, `:95S::ALTE//VISD/…` |
| 305 | TRADE | TVLK→VSDC | Confirm/reject trade settlement obligation (response to MT518) | `:77E::PROC//TRADE`, `:25D::STAT//CONF`/`REJT`, `:20C::PREV//`(MT518) |
| 308 | CASH | TVLK→VSDC | Allocation confirm (cash/quota to investor) — sample 4227.fin + dien_mau_bond.xlsx | `:23G:NEWM`, `:98A::PREP//` |
| 309 | CASH | TVLK→VSDC | **Revoke** allocation (pairs with 308) — `:20C::RELA//`(ref 308), `:25D::IPRC//REJT`, `:70D::REAS//` |
| 000 | ERRTRADE | VSDC→TVLK | Invalid trade result (cancel + reason) | `:23G:CANC`, `:20C::TRRF//`, `:70D::REAS//` |

## Notes & gotchas

- **Legacy reuse:** 001/002/003/005/006 are unchanged core-TPRL; 007/008/100/301/303/305 are carbon-new; 010/011/112/116 are hybrid.
- **Project blocker (memory):** `HSC_STP` `TemplateConfirmPaymentObligation` hardcodes legacy `:12:222` + BIC `VSDSVN01`; carbon spec requires `:12:305` `:77E::PROC//TRADE` + `VSDSVN03`. The `222→305` mismatch is an internal-template bug, not a spec sub-code.
- **MT inbound parser coverage** (`HSC_STP` `stp.Parse()`): O598/O518/O544/O546/O548/O564/O567/O568/I598 parsed; **O508/O900/O910 have no parser** — files retry then drop.
- **Sub-code 308 vs 010:** spec lists 010/011; real sample uses 308 (+309 revoke, paired) — treat as allocation; confirm with VSDC.
- **`:70D::REGI//` = investor ACCOUNT NUMBER** (số tài khoản, NOT clientId) — appears in MT518/544/546 settlement parties. Common bug: matching REGI against clientId fails.
- **Authoritative sample doc** (shared Bond+Carbon): `documents/dien_mau_bond.xlsx` (alias in Carbon memory: "Bộ điện mẫu Carbon STP.xlsx" — same file) — named "bond" but content is carbon STP; 6 sheets covering account register/adjust, listing/delist, custody deposit/withdraw, settlement flow, allocation. Extracted text: `documents/tmp_extract/dien_mau_bond.txt`. Path relative to each project root (`c:\_project_git` and `c:\_project_cabon`).

See `vsdc-stp-flows.md` for which messages each business flow produces, `swift-mt-tags.md` for tag formats, `mt-samples.md` for annotated real messages.