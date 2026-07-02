# Bond vs Carbon — How Each Uses G3

Both BondOMS and Carbon-OMS call the same G3 back-office core, the same way (direct SOAP `messageTransfer`). Carbon-OMS was **forked from BondOMS**, so the G3 integration code is near-identical — but the products differ. This file maps the differences so porting between them is safe.

## Same vs different — at a glance

| Aspect | Bond / TPRL (BondOMS) | Carbon (Carbon-OMS) |
|---|---|---|
| Transport | Direct SOAP `messageTransfer` | Same (forked code) |
| SOAP op `Type` values | `CreateCashHold` / `CreateInstrumentHold` / `CreateCashRelease` / `CreateInstrumentRelease` / `CreateAccountContract` | Same set |
| Wrapper funcs | `makeCashHold`, `makeStockHold`, `makeCashRelease`, `makeStockRelease`, `makeTransactionBond`, `makeG3Order` | `makeCashHold` (others inherited from fork; `OMSPrivateBondApi` package not renamed) |
| Instrument `MARKETID` | `TPRL` | Carbon market id (new market + "Hạn ngạch"/"Tín chỉ" instrument types set up in G3 core) |
| Namespace (REST view) | n/a (SOAP only) | reuses `/equity/cash/*` + `/equity/instrument/*` — **no `/carbon/*` namespace** |
| Account format | `\d{3}[A-Z]{3}_[A-Z]{2}` (10-char base, `_BP` suffix) | `REGEX_VALIDATE_BP_ACCOUNT` env, default `^[a-zA-Z0-9]{1,}$`; `<id>_BP` allowed |
| `VALUEDATE` source | `time.Now()` (legacy) | `ENABLE_G3_DATE=true` → auto-fetch core business date |
| Kill switch | none documented | `ENABLE_G3=false` skips all G3 calls |
| Business date refactor | not applied | `g3Date()` in `g3-date.go` feeds all 9 G3 date touch-points |

## Bond — G3 call sites

BondOMS calls G3 **in-process only** — never via NATS/Middleware. After a match, settlement continues through HSC_STP (SWIFT MT) but that does NOT trigger further G3 hold/release (release already done at match).

| Action | G3SB `Type` | Wrapper | File |
|---|---|---|---|
| Hold cash (Buy) | `CreateCashHold` `HOLDTYPE=D` | `makeCashHold` | `handler-core-api.go` |
| Hold bond (Sell) | `CreateInstrumentHold` `HOLDTYPE=T` `MARKETID=TPRL` | `makeStockHold` | `handler-core-api.go` |
| Release cash | `CreateCashRelease` | `makeCashRelease` | `handler-core-api.go` |
| Release bond | `CreateInstrumentRelease` | `makeStockRelease` | `handler-core-api.go` |
| Match → contract | composite | `makeTransactionBond` | `handler-core-api.go` |
| Match / VSD allocate | composite | `makeG3Order` | `handler-api.go` |

Bond wrapper layer (`handler-api.go`): `holdAssetOrder` / `releaseAssetOrder` switch on `bidAsk` (Buy→cash, Sell→bond-qty-only); `checkAccountOrder` pre-checks via the `dbG3SB` read-only pool (skips the cash check when `orderKind=="tprl"`); `getTotalCashHold` = `price*qty*(1+fee)` (no tax). `makeG3Order`: Buy = `makeTransactionBond` then `makeCashRelease`; Sell = `makeStockRelease` then `makeTransactionBond`.

G3-call trigger points in the bond order lifecycle: NewOrder (hold client + reciprocal same-company), CancelOrder (release), AmendOrder (the "hold vừa đủ cover" sites), Match (`makeG3Order`), Accept post-match (release-root + hold-new). The *business rules* for these → `bond-trading-flow`; *where the call sites are* → `bond-monorepo-map`.

What Bond holds: **cash** on a Buy, **bond inventory (quantity)** on a Sell.

## Carbon — G3 call sites

Carbon-OMS uses the forked SOAP path. The Carbon settlement *spec* is written against the Core API Gateway REST endpoints; the live OMS code still uses direct SOAP (migration is a TODO).

Carbon G3 mapping (from the Carbon settlement BRD, REST view):

| G3 ID | REST endpoint | Purpose |
|---|---|---|
| G3.1 | `POST /equity/cash/hold` | hold cash (buyer) at order entry |
| G3.2 | `POST /equity/cash/release` | release cash after import BUY |
| G3.3 | `POST /equity/instrument/hold` | hold asset (seller) at order entry |
| G3.4 | `POST /equity/instrument/release` | release asset after import SELL |
| G3.5 | `POST /equity/instrument/deposit` (BUY) + `/equity/cash/deposit` (SELL) | import trade (account contract transaction) |

Carbon settlement call ordering after MT544/546 (allocation side):
- **Buyer (MT544):** G3.5 import BUY (`/equity/instrument/deposit`, txCode `D` or `TFDC`) → G3.2 release cash.
- **Seller (MT546):** G3.4 release assets → G3.5 import SELL (`/equity/cash/deposit`, txCode `TFO` to credit cash).

What Carbon holds: **cash** on a buy, **carbon instrument (quota/credit, quantity)** on a sell. Carbon has a single payment method "Thanh toán ngay" (T+0) — no end-of-day settlement (that was Bond).

## Contradictions between the two memory sources — and how this skill presents them

| Topic | Bond memory says | Carbon memory says | Resolution |
|---|---|---|---|
| Transport | Direct SOAP, g3sb-api.md v1.09; spec describes mandatory ZIP+Base64 | Direct SOAP via `G3SBApi`, *and* a REST Core API Gateway exists | Both true at different layers — OMS code = direct SOAP; Core API Gateway = REST front for the same G3. SOAP is current code; REST is the future target. Documented in `soap-transport.md`. |
| Cash hold op name | g3sb-api.md names `CreateCashDepositTransaction` / `CreateCashWithdrawalTransaction` | code uses `Type="CreateCashHold"` / `CreateCashRelease` | Both kept; `CreateCashHold` is canonical for current code (it is what the SOAP `Type` literally is). Spec names noted in `api-operations.md`. |
| ZIP encoding | spec: "mandatory ZIP + Base64" | code: `<ZIP PlainTextMode="Y">` (no compression) | Both presented — `PlainTextMode="Y"` = current reality, `="N"` = the compressed mode the spec mandates. See `soap-transport.md`. |
| Amount precision | g3sb-api.md: 2 decimals `0.01`–`999,999,999.99` | code: integer VND `%.0f` | Both noted — spec allows decimals, VND has no sub-unit so code sends integers. `api-operations.md` validation table. |
| Account type encoding | `DOMIND/FORIND/DOMCORP/FORCORP` | numeric `registrationType` 3/4/5/6 | Same four investor categories, different encoding per transport. `api-operations.md` §1. |
| "Wrapper functions" | an earlier bond memory claimed G3SB has wrapper funcs | corrected: G3SB is raw SOAP, wrappers are OMS-side | Resolved — Rule 7 in `integration-rules.md`; wrappers are BondOMS/Carbon-OMS Go code. |

## Porting G3 code Bond → Carbon (and back)

- The SOAP envelope, `Type` values, and `HOLDTYPE` codes port directly.
- **Do change:** `MARKETID` (`TPRL` → carbon market id), account-validation regex, and add `ENABLE_G3` / `ENABLE_G3_DATE` handling (Carbon has them; legacy BondOMS does not).
- **Carbon-specific:** `g3Date()` business-date auto-fetch — port it *to* Bond if Bond hits the same UAT date-drift problem; it is not Carbon-only by design, just Carbon-first.
- Carbon-OMS still carries the legacy package name `OMSPrivateBondApi` and many `bond*` symbols — do not assume a `carbon*` name exists; grep.
- Settlement differs fundamentally — Carbon is T+0 single-payment; do not port Bond's end-of-day settlement G3 sequencing.