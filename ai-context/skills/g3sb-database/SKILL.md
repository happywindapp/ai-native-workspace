---
name: g3sb-database
description: HSC G3SB database — direct SQL knowledge for the G3 Securities Back-office SQL Server DB. Covers connection setup (sqlserver driver, ReadOnly intent, dbG3SB vs dbG3FB), the table naming convention (MC*/TSB*/TC*/HC*/BC*/MSB*/SX* modules), core table structures (MCClient, MCAccount, MCInstrument, TSBAccountContract, TSBAccountInstrumentMovement, TCAccountTransaction, HCAccountCash, HCBusinessDateToSystemTime, MCFeeNature), the State maker-checker codes (A/D/PU/PI/PD), no-FK logical-join rules, and the real production query patterns used by BondOMS, Carbon-OMS, CoreApiGateway and HSC_STP (business-date fetch, client/account lookup, portfolio balance from movements, fee master, margin/status, VSD mapping). Use when writing or debugging a SQL query against G3SB, joining G3 tables, reading account/contract/cash/instrument balances, computing hold/release pending quantity, or onboarding to the G3SB schema. Triggers: G3SB, G3 database, G3SB SQL, dbG3SB, MCAccount, MCClient, MCInstrument, TSBAccountContract, TSBAccountInstrumentMovement, TCAccountTransaction, HCAccountCash, HCBusinessDateToSystemTime, business date, hold release movement, VCClient, VCAccount, query G3, G3 schema, account balance, portfolio balance.
version: 1.0.0
---

# G3SB Database — Direct SQL Knowledge

## Overview

The **direct-SQL** skill for **G3SB** (*G3 Securities Back-office*), HSC's core securities DB on SQL Server. This is the companion to `g3-core-integration` (which owns the SOAP/G3 API call surface) — **this skill owns reading G3SB tables directly with SQL**. Four HSC services query G3SB directly: **BondOMS** (`C:\_project_git\BondOMS`), **Carbon-OMS** (`C:\_project_cabon\Carbon-OMS`), **CoreApiGateway** (`C:\_core_api_gateway`), and **HSC_STP** (both git/carbon repos). Everything else goes through the G3 SOAP API.

> **As-of:** schema audited 2026-06-12 from UAT `192.168.21.30:1433` (hostname `1CEU-DBG3-PR01` — "PR01" looks production, confirm env first). G3 is a third-party AFE product (no source). Symbol/line drift — grep the function name before trusting a location. Full table catalog: `references/schema-reference.md`.

> ⚠️ **READ-ONLY.** G3SB is a live core DB. Never run INSERT/UPDATE/DELETE/DDL — only SELECT + metadata. (Matches the user's global read-only DB rule.)

## Database facts

- **G3SB** ≈ 432 GB, `E:\MSSQL\G3SB.mdf`. 527 business tables (`dbo`) + 24 CDC (`cdc`). 1.917 views. **0 foreign keys.** Only 6 stored procs — logic lives in the app tier.
- Two physically separate G3 DBs with **overlapping table names**:
  - `dbG3SB` → equity / securities back-office (this skill).
  - `dbG3FB` → derivatives / futures (e.g. `VCBAccountCash`, `HCBusinessDateToSystemTime` exist in both — different server, different data).

## Connection (Go, `sqlserver` driver)

Real setup from `Carbon-OMS/conn-database.go` (BondOMS identical):

```go
queryG3SB := url.Values{}
queryG3SB.Add("ApplicationIntent", "ReadOnly")          // hits a read replica — read-only by design
queryG3SB.Add("database", os.Getenv("G3SB_Connection_Database"))
u := &url.URL{
    Scheme:   "sqlserver",
    User:     url.UserPassword(os.Getenv("G3SB_Connection_Username"), os.Getenv("G3SB_Connection_Password")),
    Host:     fmt.Sprintf("%s:%s", os.Getenv("G3SB_Connection_Server"), os.Getenv("G3SB_Connection_Port")),
    RawQuery: queryG3SB.Encode(),
}
dbG3SB, _ = sql.Open("sqlserver", u.String())
dbG3SB.PingContext(ctx)
```

- Env vars: `G3SB_Connection_{Server,Port,Username,Password,Database}`. Gateway uses an analogous pair for `dbG3FB`.
- Parameters are **named-ordinal**: `@p1`, `@p2`, … → `dbG3SB.Query(sql, arg1, arg2)`. For `IN (...)` build `@p1,@p2,…` dynamically (see `getAccountStatus`).
- Dates come back as RFC3339-ish strings (`2026-05-22T00:00:00Z`) — strip the time part before use (see `normalizeG3Date` in `g3-date.go`). Scan nullable cols into `sql.NullString`.

## Naming convention = module (decode any table by prefix)

| Prefix | Meaning | Examples |
|---|---|---|
| `MC*` | Master / config | MCClient, MCAccount, MCInstrument, MCMarket, MCFeeNature, MCUser |
| `TSB*` | Securities-book transactions | TSBAccountContract, TSBAccountInstrumentMovement, TSBAccountEntitlement |
| `TC*` | Cash + ledger | TCAccountTransaction (298M, biggest), TCAccountCashMovement, TCAuditLog |
| `HC*` | End-of-day balance snapshot (`ValueDate` in PK) | HCAccountCash, HCAccountLocationInstrument, HCBusinessDateToSystemTime |
| `BC*` | Accruals (interest/fee) | BCAccountAccruedInterest, BCAccountAccruedCustodianFee |
| `MSB*` | Corporate-action master | MSBEntitlement |
| `SX*` | Gateway message queues (VSD/bank) | SXGatewayRequestLog, SXSynchronizer* |
| `XC*` `XSB*` `WSB*` `WC*` | exchange/snapshot, workflow batch | XCAccount, WSBAccountContractSettlement |
| `VC*` `VSB*` | **Views** (denormalized read layer) | VCClient, VCAccount, VSBBAccountInstrument |

**Rule of thumb:** read through `VC*`/`VSB*` **views** when you can (denormalized, app-blessed); hit `MC*`/`TSB*`/`TC*` **base tables** for masters or raw movements.

## `State` — maker-checker lifecycle (CRITICAL filter)

`State nchar(4)` is on 354 tables and is **usually part of the PK**, so the same business key has several rows. Always filter it.

| State | Meaning |
|---|---|
| `A` | Active / approved (the live row) |
| `D` | Deleted (logical — never physically removed) |
| `PI` | Pending Insert (new, awaiting checker) |
| `PU` | Pending Update (edit awaiting checker) |
| `PD` | Pending Delete (delete awaiting checker) |

- Reading current truth → `WHERE State = 'A'`.
- Reading in-flight/unsettled work (e.g. pending movements) → include `('PI','PU','A')` as the movement query does.

## No FKs — join by column-name convention

Relationships are conventions, not constraints. Key join columns: `AccountID` (154 tables), `MarketID`+`InstrumentID` (always paired, 88), `CurrencyID` (98), `ClientID`, `ContractID`, `TransactionID`. Central axis: **`MCClient (1) ──< MCAccount (n)`**; `MCAccount.AccountID` is the hub of nearly every transaction/balance table.

> **Gotcha — client lookup is NOT by AccountID.** `VCClient` joins to `VCAccount` via **`CustodyID`**, and you filter on `VCAccount.AccountID`. `VCClientContact` joins via `ClientID = CustodyID`. See query patterns.

## Always use `WITH (NOLOCK)` on hot tables

TCAccountTransaction, TSBAccountContract, TSBAccountContractFee, HC* are huge and write-hot. All HSC code reads them `WITH (NOLOCK)` to avoid blocking the core. Do the same.

---

## Production query patterns (copy-ready, from the 4 services)

### G3 business date — the time anchor for everything
```sql
SELECT TOP(1) businessDate FROM HCBusinessDateToSystemTime WHERE EndTime IS NULL ORDER BY StartTime DESC
```
Used by `g3-date.go` (BondOMS/Carbon-OMS) and gateway `getBusinessDate`. Cache it (BondOMS caches 60s). For derivatives ask `dbG3FB` with the same SQL. Empty result ⇒ G3 hasn't opened the day.

### Client + account info (note the CustodyID join)
```sql
SELECT VCClient.ClientID, VCClient.Name, VCClient.RegistrationType, VCClient.IDType, VCClient.IDNumber,
       VCClient.DateOfBirth, VCClient.NationalityID, VCClient.TradingCode, VCClient.RegistryDate,
       VCAccount.BranchID, VCAccount.AEID
FROM [G3SB].[dbo].[VCClient] AS VCClient
LEFT JOIN [G3SB].[dbo].[VCAccount] AS VCAccount ON VCAccount.CustodyID = VCClient.CustodyID
WHERE AccountID = @p1
```
Contact: `VCClientContact` (Address1, SMSNumber, EMailAddress) `LEFT JOIN VCAccount ON VCAccount.CustodyID = VCClientContact.ClientID WHERE AccountID = @p1`.

### Portfolio balance = view balance + pending movements
Available stock = settled balance from `VSBBAccountInstrument` **plus** pending deltas computed from `TSBAccountInstrumentMovement`. The pending sign rule (from `portQueryString`, the canonical hold/release math):
```
sign by MovementType/HoldType/AutoReleaseDate:
  D & HoldType NULL                                   → +1   (deposit in)
  W                                                   → -1   (withdraw)
  T                                                   → -1   (transfer out)
  D & HoldType NOT NULL & AutoReleaseDate NOT NULL    → +1
  H & HoldType NOT NULL & AutoReleaseDate NULL & State='PI' → -1  (new hold)
  H & HoldType NOT NULL & AutoReleaseDate NOT NULL    → +1
× (State='A' ? -1 : +1)
× (HoldType NOT NULL & AutoReleaseDate NOT NULL & AutoReleaseQuantity NOT NULL
     ? AutoReleaseQuantity : Quantity - COALESCE(ReleasedQuantity,0))
filter: tx.State IN ('PI','PU','A'); pending rows joined on State IN ('PI','PU','PD')
```
Full SQL in `references/schema-reference.md` §query-patterns. This is *the* example of how hold/release on `TSBAccountInstrumentMovement` maps to a real balance.

### Cash / margin excess
```sql
SELECT ExcessEquity FROM [G3SB].[dbo].[VSBAccountConsolidatedMarginSummaryEQD] WHERE AccountID = @p1
-- gateway equity cash:
SELECT [Settled],[AvailableBalance],[MaximumAvailableBalance],[AccruedCreditInterest],[AccruedDebitInterest]
FROM [G3SB].[dbo].[VCBAccountCash] WHERE AccountID = @p1
-- derivatives (dbG3FB): SELECT [ExcessOrDeficit] FROM [VCBAccountCash] WHERE AccountID = @p1
```

### Fee master (FeeNature → MarketFee → rate timetable)
```sql
SELECT DISTINCT MCFeeNature.NameEx, VCCalculationMethodTimetableDetail.Rate1, VCCalculationMethodTimetableDetail.EffectiveDateFrom
FROM [G3SB].[dbo].[MCFeeNature] AS MCFeeNature WITH (NOLOCK)
LEFT JOIN [G3SB].[dbo].[MCMarketFee] AS MCMarketFee ON MCFeeNature.FeeNatureID = MCMarketFee.FeeNatureID
LEFT JOIN [G3SB].[dbo].[VCCalculationMethodTimetableDetail] AS d ON MCMarketFee.CalculationMethodID = d.CalculationMethodID
WHERE MCMarketFee.MarketID = @p1
ORDER BY VCCalculationMethodTimetableDetail.EffectiveDateFrom ASC
```

### Margin contract, account status, VSD mapping (gateway)
```sql
SELECT AccountID, MarginContractNumber, MarginRenewDate FROM [G3SB].[dbo].VCAccount WHERE AccountID = @p1 AND AccountTypeID='M'

SELECT AccountID, StatusCode, sc.Name AS StatusName
FROM [G3SB].[dbo].[VCAccountStatus] s
LEFT JOIN [G3SB].[dbo].[VCAccountStatusCode] sc ON s.StatusCode = sc.AccountStatusCode
WHERE AccountID IN (@p1,@p2,…) AND s.State = 'A'

SELECT ClearingDomain, GatewaySendStatus FROM MCAccountVSDMapping WHERE State='A' AND AccountID = @p1
```
`GatewaySendStatus` recurs across movement/entitlement tables = status of the VSD/bank message for that record.

### Day's contracts + per-contract fee (Carbon, generalizes to bond)
```sql
SELECT LTRIM(RTRIM(ContractID)) ContractID, TradeDate, LTRIM(RTRIM(TSBAccountContract.InstrumentID)) InstrumentID,
       Price, Quantity, LTRIM(RTRIM(AccountID)) AccountID,
       REPLACE(REPLACE(BuySell,'B','buy'),'S','sell') BuySell
FROM [TSBAccountContract]
LEFT JOIN [VCInstrument] ON TSBAccountContract.InstrumentID = VCInstrument.InstrumentID
WHERE TradeDate = @p1 AND InstrumentTypeID IN (…) AND VCInstrument.MarketID = @p2
-- fees: SELECT … FROM [VSBLedgerGenerationAccountContractByCashSettleDate] WHERE AccountID=@p1 AND TransactionID=@p2 AND TradeDate=@p3
```
> `nchar`/`nvarchar` keys are space-padded — `LTRIM(RTRIM(...))` when comparing/returning. Filter Unicode literals with `N'...'`.

---

## Gotchas checklist

- `State='A'` filter — forget it and you get pending/deleted dupes.
- `WITH (NOLOCK)` on TC*/TSB*/HC* — they're hot.
- Client lookup joins on **CustodyID**, not AccountID.
- `MarketID` + `InstrumentID` is a **composite** key — never join `InstrumentID` alone.
- `dbG3SB` ≠ `dbG3FB`: same table names, equity vs derivatives. Pick the right handle.
- Trim space-padded `nchar` keys; use `N'...'` literals.
- HC* balance tables split header (key + `RecordID`) and `…Record` (the measures) — join on `RecordID`.
- Dates from the driver carry a time component — normalize to `2006-01-02`.

## Which service to grep for a working example

| Need | File |
|---|---|
| Connection / pool | `Carbon-OMS/conn-database.go`, `BondOMS/conn-database.go` |
| Business date + cache | `*/g3-date.go` |
| Client/account/portfolio/fee query builders | `*/handler-common.go` |
| Account fee defaults | `*/handler-core-api.go` |
| Instrument master, cash, margin, status, VSD | `CoreApiGateway/func-common.go`, `func-cash.go`, `func-core-equity.go` |
| Client via GORM | `HSC_STP/internal/infra/repo/account_info.go` |

See `references/schema-reference.md` for the full column-level table catalog and the complete portfolio-balance SQL.
