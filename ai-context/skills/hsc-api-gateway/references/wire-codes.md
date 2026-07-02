# API Wire Codes (Equity + Derivatives)

Most write endpoints in CoreApiGateway accept short string enum codes (single letters or 2-4 char strings) instead of human-readable values. These codes are part of the **wire contract with backend systems (G3SB / G3FB / XML)** and cannot be changed without a coordinated migration. They are documented in `docs/Core API Gateway.postman_collection.json` and the xlsx — **not** in the Go source.

> **Rule:** When you see a single-character string field in a request struct or DB table, do NOT rename, normalize, or "improve" it. When generating a sample request, use these codes verbatim. The xlsx/Postman are the source of truth.

## Order placement — `/equity/order/new`

| Field | Codes |
|---|---|
| `bidAsk` | `B` = Buy, `S` = Sell |
| `origin` | channel/origin code, one of: `X` `I` `O` `M` `F` `E` `T` `K` `B` `P` `R` (meanings live in the trading core; gateway passes through) |
| `dtrade` | bool — true if T+0 day trade |
| `bothSideForeigner` | `T` or `Y` when both counterparties are foreign investors |

## Equity instrument operations

| Field | Codes |
|---|---|
| `holdType` (hold/release) | `T` = Temporary hold, `B` = Block, `7` = Taxable Bonus |
| `transactionCode` (deposit) | `D` = Deposit, `DC` = Conditional, `TFDC` = Transfer From Deposit Center (VSD), `TFO` = Transfer From Outside |
| `transactionCode` (withdrawal) | `W` = Withdrawal, `TTDC` = Transfer To Deposit Center, `TTO` = Transfer To Outside |

## Derivatives cash operations
Different vocabulary from equity:

| Field | Codes |
|---|---|
| `transactionCode` (deposit) | `COL_DEP`, `COL_DEP_CCP`, `CD`, `CFO`, `D`, `OCD` |
| `transactionCode` (withdrawal) | `CTO`, `CW`, `OCW`, `COL_WITH` |

## Account creation — `/equity/account/create`, `/derivatives/account/create`

| Field | Codes |
|---|---|
| Equity `accountTypeId` | `X` = Brokerage (cash), `M` = Margin |
| Derivatives `accountType` | `H` = Home, `I` = Market Maker, `M` = Normal, `O` = Omnibus |
| Equity `accountStatus` | `0` Normal · `1` Suspended · `4` Closed · `6` No Trading · `8` Prepare to Close · `15` Close VSD · `16` Shareholder Only · `17` Transfer VSD · `P` Disallow OTP |
| Derivatives `accountStatus` | `000` Normal · `040` Closed · `050` Suspended (3-digit padded, not 0/1/…) |
| `registrationType` | `3` Local Retail · `4` Foreign Retail · `5` Local Institutional · `6` Foreign Institutional · `7` Portfolio (derivatives) · `9` Government (derivatives) |
| `idType` | `1` SID · `2` Passport · `4` Other ID · `5` Business ID |
| `gender` | `M` / `F` |
| `methodOfReceivingStatement` (derivatives) | `E` electronic / `M` mail |
| `referType` (derivatives) | `1` / `2` |
| Boolean flags | `notification`, `vsdFlag`, `foreignBankFlag`, `cashCustodyFlag`, `stockCustodyFlag`, `fatcareRegistrationFlag`, `sellOnlyFlag`, `buyOnlyFlag`, `usingPhoneCenter`, `autoApprovalFlag` — all `Y` / `N` |

## ORS (rights) exercise — `/equity/entitlement/ors/create`

| Field | Codes |
|---|---|
| `autoApproval` | `T` / `F` **as strings** — NOT `Y`/`N` like other flags. Watch this inconsistency. |
| `processAction` | `H` = hold (block cash), `W` = deduct (withdraw cash) |

## Client status — `/equity/client/update-status`

| Field | Codes |
|---|---|
| `statusCode` | `N` = NORMAL, `C` = CLOSED |