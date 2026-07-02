# G3 Core / G3SB — API Operation Catalog

G3SB operations come in two forms in HSC's stack:
- **SOAP form** — what the OMS code (Bond + Carbon) actually calls today. Inner payload `<REQUEST Type="...">`. Spec: `g3sb-api.md` v1.09 (N2N-AFE 2019-11-14).
- **REST form (Core API Gateway)** — `{{coreAPIHost}}` under the `/equity/*` namespace. Carbon settlement is *designed* against this, but OMS code currently uses direct SOAP. Migration SOAP→REST is a TODO.

Transport details (envelope, encoding, auth) → `soap-transport.md`. Error codes → `error-codes.md`.

> Two memories describe G3 differently — the **Bond `g3sb-api.md` v1.09** SOAP spec and the **Carbon `Core API Gateway` Postman collection** REST spec. Field names below show both where they differ; see `bond-vs-carbon-usage.md` for the reconciliation.

## 1. Account operations

| Operation | SOAP `Type` | REST (Core API GW) | Key inputs | Returns |
|---|---|---|---|---|
| Create account | `CreateAccount` | `POST /equity/account/create` | `registrationType` (3 Local Retail · 4 Foreign Retail · 5 Local Inst · 6 Foreign Inst), `accountTypeId` (X Brokerage · M Margin), investor/KYC fields | account no + status |
| Update account | `UpdateAccount` | `POST /equity/account/update` | partial fields | status |
| Query account | `GetAccount` / `QueryAccount` | `GET /equity/account/status?accountNo=a,b` | `AccountNo` (REST: comma list) | AccountNo, AccountType (`DOMIND`/`FORIND`/`DOMCORP`/`FORCORP`), Status (`ACTIVE`/`INACTIVE`/`FROZEN`), CashBalance, CashOnHold, InstrumentBalance[], ContractList[], LastUpdateTime |
| Interest setting | — | `POST /equity/account/interest-setting` | accountNo + currencyId + interestClassId | status |
| Fee setting | — | `POST /equity/account/fee-setting` | accountNo + marketId + feeNatureId + calculationMethodId | status |

> Bond `g3sb-api.md` v1.09 names account types `DOMIND/FORIND/DOMCORP/FORCORP`; Carbon Core-API uses numeric `registrationType` 3/4/5/6 — same four investor categories, different encoding.

## 2. Cash hold / release

| Operation | SOAP `Type` | REST | Inputs | Returns |
|---|---|---|---|---|
| Cash hold | `CreateCashHold` | `POST /equity/cash/hold` | `VALUEDATE`, `TRANSACTIONREFERENCE` (unique), `ACCOUNTID`, `HOLDTYPE=D`, `AMOUNT TYPE="DECIMAL"`, `REMARK`, `AUTOAPPROVALFLAG=Y` | success → TransactionRef GUID + `ACCEPTED`; fail → ErrorCode + ErrorMessage |
| Cash release | `CreateCashRelease` | `POST /equity/cash/release` | original hold reference, `ReleaseAmount` (partial OK), `Reason` | `RELEASED` + RemainingAmount; or ErrorCode + RemainingAmount |
| Cash deposit | — | `POST /equity/cash/deposit` | accountNo, valueDate, transactionReference, amount, remark, autoApprovalFlag | status |
| Cash withdrawal | — | `POST /equity/cash/withdrawal` | same shape as deposit | status |
| Cash balance | — | `GET /equity/cash/balance-g3?accountNo=` | accountNo | balance |

- `g3sb-api.md` v1.09 also names cash hold ops `CreateCashDepositTransaction` / `CreateCashWithdrawalTransaction` (both share the deposit schema: `TransactionNo`, `AccountNo`, `Amount`, `Currency="VND"`, `ValueDate`, `Description`). The OMS code uses the simpler `CreateCashHold` / `CreateCashRelease` `Type` values — treat `CreateCashHold` as canonical for current code.
- **HoldType `D`** = cash hold (Default/Temporary). Amount sent as integer VND (`%.0f`).

## 3. Instrument (stock / bond) hold / release

| Operation | SOAP `Type` | REST | Inputs | Returns |
|---|---|---|---|---|
| Instrument hold | `CreateInstrumentHold` | `POST /equity/instrument/hold` | accountNo, `MARKETID` (Bond: `TPRL`; Carbon: carbon market id), instrumentId, quantity, `HOLDTYPE` (`T` Temp / `B` Block / `7` Taxable Bonus), transactionReference, autoApprovalFlag | status / ref |
| Instrument release | `CreateInstrumentRelease` | `POST /equity/instrument/release` | same | status |
| Instrument deposit | — | `POST /equity/instrument/deposit` | + `transactionCode` (`D` / `DC` / `TFDC` / `TFO`) | status |
| Instrument withdrawal | — | `POST /equity/instrument/withdrawal` | + `transactionCode` (`TTDC` / `TTO` / `W`) | status |
| Instrument master/alias/overriding | — | `GET /equity/instrument/{master-info,alias-name,overriding-info?accountNo=}` | — | instrument metadata / balance by instrument |

- Instrument hold uses **quantity only** — no price (price is a cash-hold concern).
- Bond `makeStockHold` sends `HOLDTYPE=T`, `MARKETID=TPRL`.

## 4. Contract / import-trade

| Operation | SOAP `Type` | REST | Inputs | Returns |
|---|---|---|---|---|
| Create account contract | `CreateAccountContract` | (Core API: import-trade endpoints) | `AccountNo`, `ContractType` (`TPRL`/`REPO`/`OUTRGHT`), `EffectiveDate`, optional `TerminationDate`, `FeeSchedule[]` (buy/sell/hold %), optional `RiskLimit` | ContractNo GUID + `ACTIVE` |
| Import BUY trade | (composite) | `POST /equity/instrument/deposit` (txCode `D`/`TFDC`) | settlement allocation inputs | status |
| Import SELL trade | (composite) | `POST /equity/cash/deposit` (txCode `TFO`) | settlement allocation inputs | status |

- **Fees are embedded** in `CreateAccountContract.FeeSchedule[]` — there is no separate fee-sync operation. (Corrects an earlier "separate fee op" note.)
- Bond `makeTransactionBond` is the composite "match → contract" call; `makeG3Order` handles match / VSD-allocate.

## 5. Query / statement / refdata

| Operation | SOAP | REST | Notes |
|---|---|---|---|
| Cash statement | `QueryCashStatement` | `GET /equity/statement/cash?accountNo=&startDate=&endDate=` | filter `DEPOSIT/WITHDRAWAL/HOLD/RELEASE/FEE` → `{Date, Amount, Type, Status, Description}[]` |
| Instrument statement | `QueryInstrumentStatement` | `GET /equity/statement/instrument?...` | optional ISIN filter → `{SecurityCode, SecurityName, Quantity, Value, Status}[]` |
| History | — | `GET /equity/history/{order,trade,balance,last-week-balance,previous-balance}?accountNo=&startDate=&endDate=` | — |
| Refdata | — | `GET /equity/{business-date,brokers,interest-class,account-class,account-type}` · `GET /common/{holiday,branch}` | `business-date` is the canonical G3 business date source |

## 6. Order operations (Core API Gateway, REST)

| Operation | REST | Purpose |
|---|---|---|
| New order | `POST /equity/order/new` | create order |
| Amend order | `POST /equity/order/amend` | modify order |
| Cancel order | `POST /equity/order/cancel` | cancel order |

## 7. OTP

| Operation | REST | Inputs |
|---|---|---|
| Request OTP | `POST /otp/request` | uid, expiredTime (180s), maxResend (3), maxFail (3) |
| Verify OTP | `POST /otp/verify` | uid, requestId, otp |

## Validation rules (g3sb-api.md v1.09)

| Field | Rule |
|---|---|
| Account no | Bond format `\d{3}[A-Z]{3}_[A-Z]{2}` (e.g. `011ABC_BP`). Carbon-OMS uses regex env `REGEX_VALIDATE_BP_ACCOUNT` (default `^[a-zA-Z0-9]{1,}$`, `_BP` suffix allowed). |
| Amount | g3sb-api.md spec: 2 decimals, `0.01`–`999,999,999.99` VND. OMS code: sent as **integer VND** (`%.0f`) — VND has no sub-unit. |
| ISIN | `^[A-Z]{2}[A-Z0-9]{9}[0-9]$` |
| Date | `YYYY-MM-DD` (ISO 8601). `VALUEDATE` must equal G3 core business date — see `integration-rules.md`. |
| TxID / `TRANSACTIONREFERENCE` | unique per day per account; OMS pattern `yyyyMMddHHmmss{randSeq}`. Drives idempotency. |