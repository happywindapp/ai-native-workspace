# Carbon vs Bond (TPRL) Differences

Carbon-OMS is a fork of `OMSPrivateBondApi` (BondOMS); Carbon-Terminal/Middleware are clones of the Bond stack. Most patterns are identical — but many points differ critically. Knowing them prevents copy-paste bugs.

## 1. Domain differences

| Element | TPRL (Bond) | Carbon |
|---|---|---|
| Trading tables | 1 TPRL table | **2 tables**: Hạn ngạch (HN) + Tín chỉ (TC) |
| Investor types | 2: NĐTCN + NĐT TT | **3**: EMIT (HN or TC) / PROJ (TC only) / ORGA (TC only) |
| Codes | bond ISIN (issuer) | carbon ISIN issued by DCC (`VN000000CARB` pattern) |
| Band/Tick/Lot | per-instrument | HN/TC differ, change daily during testing — **load from Core G3, never hardcode** |
| Payment method | 2: T+0 + T+n (end-of-day) | **only 1: T+0** ("Thanh toán ngay"); FE `paymentOptions` keeps only value `"1"` |

## 2. Flow differences

- **Deposit/withdraw:** TPRL is TVLK-initiated; Carbon is **DCC-initiated** (Bộ NN&MT) — TVLK only receives MT544/546. (Phase 1 not wired — D2 deferred.)
- **Proprietary-error correction:** TPRL has it; Carbon Phase 1 does **NOT** (QC ver03 Điều 14 exists but is Phase-2 implementation).
- **Payment removal:** Carbon treats it as a kind of "post-trade error correction" with an early cut-off (15:30 submit docs → 15:45 VSDC processes → 16:30 removed via MT518 `23G:CANC`).

## 3. Integration differences

- **NHTT:** VCB (TPRL) → **VietinBank** (Carbon — IBM API Connect + JWT signing). OMS has not migrated the code yet (see `monorepo-map.md` + memory `vietinbank-api-integration`).
- **VSDC STP:** BIC `VSDSVN03` (ending `03` = carbon; never the TPRL BIC); BRID balance report `0009` (TPRL `0008`); new Carbon MT598 sub-types **007/008/100/112/116/301/303/305/010/011**; extra qualifiers `:22F::TPTY` + `:22F::ACTP` on MT598.301. Shares the STP gateway with TPRL (lane resolver distinguishes — see `stp-end-to-end-flow.md`).
- **Market data:** HNX InfoGate carries 2 new carbon FIX messages (CBS/CBB) — see `integrations.md`.

## 4. Regulation

NĐ 156/2020 (securities) → **NĐ 29/2026/NĐ-CP** (carbon exchange) + Quy chế carbon VSDC Ver03 + TT 11/2026/TT-BNNMT (credit freeze/unfreeze).

## 5. Rules for porting BondOMS → Carbon

1. **Do NOT copy-paste TPRL business logic without review** — settlement / cut-off / account format differ.
2. **Always check VSDC BIC = `VSDSVN03`** when composing a Carbon STP message.
3. **Replace VCB with VTB** — JWT signing pre-request, endpoint `/remittance-add`.
4. **Add `:22F::TPTY` + `:22F::ACTP`** to MT598.301 register-account.
5. **Test all 3 subject types** EMIT/PROJ/ORGA (vs Bond's 2).
6. **Load Tick/Lot/Band from Core G3** — never hardcode per-instrument.
7. **No "proprietary-error correction" in Phase 1** — skip that flow when porting.
8. **G3 reuses the `/equity/*` API** with Carbon `marketId` + `instrumentId` (already created in Core).

## BondPlus removal (2026-04-24)

Bond Plus (`orderKind='bondplus'`) was fully removed from all 3 Carbon-* sources (Middleware, OMS, Terminal). Only the TPRL flow (`orderKind='tprl'`) remains.

- **Do not re-introduce removed code:** route `/internal/*`, `pages/bondplus.tsx`, folder `BondPlus/`, MW `addQueryInternalListCommands`, validator `commandBondplusAddNewValidate`.
- When touching `order_kind` column / `OrderKind` field → default/force `'tprl'`. MW auto-defaults in `controllers/command.js`.
- **`DefaultAccountCLassId = "BOND_PLUS"`** in OMS `datastruct/constant.go:49` is **NOT** a Bond Plus feature — it is an AccountClassID in the G3SB fee table. Kept pending G3 confirmation.
- **The `<id>_BP` account-input flow is separate and still supported** (2026-05-20 clarification). `_BP` account input on the order screen is not BondPlus code — do not revert it.
- Leftover pending cleanup: DB column `order_kind` + migration 000014 (kept, default `'tprl'`); `OrderKind` field in 4 `datastruct/core-api.go` structs + SQL filters.

## endDate removal (2026-05-13/14)

`endDate` (inherited from Bond) was stripped because "Carbon accounts are registered permanently".

- **Removed** from FE (`Register/index.tsx`, `CustomersList`, state/payload/UI/filter/export) and MW (`account.js` validation `end_date_invalid`, and a dangerous side-effect in `getList` that auto-nullified `accountTypeName` for accounts with `endDate < now`).
- **Kept (intentional):** DB column `endDate` (`allowNull:true`, stored null); STP forwards `end_date: null`; HSC_STP Go skips processing when empty.
- Carbon-OMS: `grep endDate|end_date|EndDate` → 0 references (fully clean).
- See `account-rules.md` "Permanent registration".

## Rebrand debt (still present — Carbon-OMS not yet renamed)

The Carbon stack still carries Bond identifiers. Decision pending: does the Carbon stack still run TPRL in parallel? If yes → keep aliases; if no → plan a tiered rename. **Ask the user before any rebrand cleanup.**

- **OMS:** go.mod module `OMSPrivateBondApi`; struct `BondCode`; column `bond_code`; `MARKETID="TPRL"` hardcoded in SOAP envelopes; fee consts `DefaultCommissionBond*`; `MarketFee="TPRL"`; swagger `@Tags Bond Order`; filter `InstrumentTypeID='CORP BOND'`; error code `accountNotEnoughBond`.
- **MW:** route group `/bonds`; model `bondCode`; validator `body("bondCode")`; `defaultOrderKind="tprl"`; error `NOT_BOND_ACCOUNT`; health string `"Bond Terminal Api Services"`.
- **STP:** route prefix `/v1/stps/private-bond/*` still present beside `/v1/stps/carbon/*`.
- **Terminal:** ~22 files still use `IBond/IBondItem`, `bondStatusMapping`, query param `bondCode`, endpoint `bonds/*` paths.