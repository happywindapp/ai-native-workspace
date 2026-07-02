# G3SB Schema Reference — Table Catalog

Column-level catalog of the G3SB core tables. PK in **bold**. Audit 2026-06-12 (`192.168.21.30`, SQL Server 2022). No FKs anywhere — joins are by column-name convention. Audit columns `CreatedBy/On, UpdatedBy/On, CheckedBy/On` exist on ~350 tables; omitted below unless relevant.

> ⚠️ No `--` comments or extended properties exist in the schema (`MS_Description = 0`). Every description here is inferred from name + type + sampled data, NOT vendor documentation. The full code-value catalog lives in `MCCode` / `MCTransactionCode`.

## Module prefixes

| Prefix | Count | Meaning |
|---|---|---|
| MC | 225 | Master / configuration |
| TSB | 60 | Securities-book transactions |
| XC / XSB | 51 / 29 | Exchange / external snapshot / intraday |
| HC | 43 | End-of-day balance snapshot (`ValueDate` in PK) |
| WSB / WC | 42 / 8 | Workflow / batch postings |
| TC | 27 | Cash + ledger + audit |
| SX | 15 | Gateway message queues (VSD/bank) |
| BC | 13 | Accruals (interest/fee) |
| MSB | 7 | Corporate-action master |
| cdc.*_CT | 24 | Change Data Capture |

## Top tables by size

| Table | Rows | Size |
|---|---|---|
| TCAccountTransaction | 298M | 47 GB |
| TSBAccountContract | 50.4M | 23.7 GB |
| SXGatewayRequestLog | 38.3M | 16.8 GB |
| HCAccountLocationInstrument | 230M | 16.4 GB |
| BCAccountAccruedInterest | 76.7M | 15.3 GB |
| HCAccountCash | 176M | 14.5 GB |
| TSBAccountContractFee | 74M | 5.6 GB |
| HCAccountMarginSummary | 76M | 5.1 GB |
| TSBAccountContractSettlement | 49.2M | 4.8 GB |

---

## Master (MC*)

### MCClient — investor (130.6K active). PK = (**ClientID** nchar30, **State**)
Identity: `Name, NameEx, FirstName/Middle/LastName, IDType, IDNumber, IDIssueDate/Place/ExpiryDate, DateOfBirth, NationalityID, TaxCode, CustodyID, TradingCode, RegistryDate`.
Classification: `RegistrationType, CustodianFlag, MutualFundFlag, AssetManagementFlag, OmnibusFlag, FATCARegistrationFlag, StaffRelatedClientFlag`.
KYC: `Occupation, Employer, AnnualSalary, SourceOfFunds, InvestmentExperience, InvestmentTimeHorizon, RiskLevel`.
Placement: `MobileNumber, BranchID, AEID, CustodianBankID`.

### MCAccount — trading account (134.4K; 134.4K active). PK = (**AccountID** nchar30, **State**)
Classification: `AccountTypeID, AccountClassID, BranchID, AEID, RegistrationType, HouseAccountFlag, MarginFacilityFlag`.
Margin: `MarginPercentageMultiplier, MarginLimit, MarginCallMultiplier, MarginForcedSellMultiplier, MarginDebitRatio, MarginEquityRatio, MaintenanceMarginRatio, MarginContractNumber, MarginRenewDate, ForceSellFlag/Date`.
Cached balances: `AvailableBalance, MaximumAvailableBalance, MarketValue, MarginValue, AssetBalance, LiabilityBalance`.
Limits: `TradingLimit, SingleOrderLimit`.
Links: `ParentAccountID, RelatedAccountGroup, StockBorrowingAccountID, FuturesAccountID, CustodyID, AccountTypeID='M'` ⇒ margin account.

### MCInstrument — instrument/product (9.6K). PK = (**MarketID**, **InstrumentID**, **State**)
Basics: `Name, BoardID, LotSize, ParValue, InstrumentTypeID, InstrumentClassID, UnderlyingInstrumentID, ListingStatus ('L'/'N'=listed), ListingDate, MaturityDate`.
Price: `ClosingPrice, ClosingPriceDate, PriceCeiling, PriceFloor, SettlementPrice`.
Margin/lending: `MarginPercentage, CollateralRatio, PledgePercentage, MaxBorrowQuantity, AllowBorrowingFlag, AllowRepoFlag`.
Settle: `TradeCurrencyID, AccountBuy/SellCashSettleDays, AccountBuy/SellInstrumentSettleDays`.
Flags: `SuspendFlag, CanShortSellFlag, CentralClearingFlag, TPlusFlag, BlackListFlag, ComplexProductFlag`.

### MCMarket — market/exchange (6). PK = (**MarketID**, **State**)
`CalendarID, TradeCurrencyID, DefaultTradeType, TradeDate, NextTradeDate, MarketClosedFlag, InHolidayFlag, VSDFlag, VSDCode, RegionID, CountryID, AccountBuy/SellCashSettleDays, AccountBuy/SellInstrumentSettleDays`.

### MCFeeNature — fee type (27). PK = (**FeeNatureID**, **State**)
`Type, Category, CurrencyType, LedgerID, AccountFeeFlag, BrokerFeeFlag, ExchangeFeeFlag, PostingFlag, RoundingMethod, RegistrationGroupID`.

### MCMarketFee — fee↔market map (93). PK = (**MarketID**, **FeeNatureID**, **State**) · `CalculationMethodID`.

---

## Securities-book transactions (TSB*)

### TSBAccountContract — buy/sell contract (50.4M). PK = (**ContractID** nchar20, **State**)
Order: `TradeDate, AccountID, AEID, BuySell ('B'/'S'), MarketID, InstrumentID, CurrencyID, Price, Quantity, Consideration, TradeType`.
Fee/broker: `InterestAmount, OverrideCommissionRate, BrokerID, BrokerCommissionAmount, BrokerSettleAmount`.
Settle: `InstrumentSettleDate, CashSettleDate, SettleQuantity, SettleAmount, SettledQuantity, SettledAmount, SettleStatus ('S'=settled/'U'=unsettled), SettleLocationID, SettleCustodianBankID, HoldForSettlementFlag`.
Group/cancel: `CancellationGroupID, CreationGroupID, MergeTradeFlag, MergeID, BatchID`.

### TSBAccountContractDetail — fill detail (50.4M). PK = (**ContractID**, **Sequence**, **State**)
`Price, Quantity, ShortSellQuantity, OrderID, ParentOrderID, TradeReference, TradeTime, ChannelID, CounterPartyID, BCAN, Exchange`.

### TSBAccountContractSettlement — settlement allocation (49.2M). PK = (**ContractID**, **SettlementID**, **State**)
`SettlementDate, SettledQuantity, SettledAmount, SettlementDifferenceAmount, SettleCashMethod, BankAccountID, BatchID`.

### TSBAccountContractFee — per-contract fee (74M). PK = (ContractID, FeeNatureID, …).

### TSBAccountInstrumentMovement — instrument move / hold-release (459K). PK = (**MovementID** nchar20, **State**)
Core: `ValueDate, AccountID, MovementType ('D'/'W'/'T'/'H'), TransactionCode, MarketID, InstrumentID, Quantity, RegisteredQuantity, CostAmount`.
Hold/release: `NeedHoldFlag, HoldType, SettleHoldType, AutoReleaseDate, AutoReleaseQuantity, ReleasedDate, ReleasedQuantity, ConvertToHoldType, ConvertedHoldQuantity`.
Settle/VSD: `SettleLocationID, SettleInstrumentMethod, SettleStatus, GatewaySendStatus, ShareholderCode, ReferenceNumber, MatchingReference`.
Totals: `TotalBoughtQuantity/Consideration/Fee, TotalSoldQuantity/Consideration/Fee, TotalCashDividend`.

### TSBAccountEntitlement — per-account entitlement (2.6M). PK = (**EntitlementID**, **TransactionID**, **State**)
`AccountID, MarketID, InstrumentID, LocationID, Quantity, RegisteredQuantity, ReinvestQuantity, SubscriptionPrice/Quantity/Amount, ExcessQuantity/Amount, SettleStatus, GatewaySendStatus`.

---

## Cash + ledger (TC*)

### TCAccountTransaction — general ledger (298M, biggest, **heap, no PK**). Logical key = `TransactionID + Sequence`
`TransactionID, Sequence, TradeDate, CashSettleDate, InstrumentSettleDate, Type, TypeID, SubType, AccountID, CurrencyID, Amount, MarketID, InstrumentID, Quantity, RealizedAmount, CostAmount, GeneralLedgerPostedFlag, ReverseState, ReverseTransactionID, LinkupType, LinkupID, ValueDate`. `Type` is a compound code (e.g. `SCDFW, CPFID, SCNBC`).

### TCAccountCashMovement — cash move / hold-release (7.46M). PK = (**MovementID**, **State**)
`ValueDate, AccountID, MovementType, CurrencyID, Amount, SettleCashMethod, SettleStatus`. `MovementType`: `D`=deposit, `W`=withdrawal, `H`=hold, `R`=release, `T`=transfer.
Hold: `NeedHoldFlag, HoldType, AutoReleaseDate, AutoReleaseAmount, ReleasedDate, ReleasedAmount`.
Bank: `BankAccountID, BankTransactionCode, From/ToBankName/Branch/AccountNumber/AccountName, TransferToAccountID, CancellationGroupID, BatchID, ProcessState`.

### TCAuditLog / TCAuditLogEntry / …EntryKey / …EntryContent — change log (220M+ content rows).

---

## End-of-day balances (HC*) — header + Record split

### HCAccountCash — daily cash balance (176M). PK = (**ValueDate**, **AccountID**, **CurrencyID**, **RegionID**) + `RecordID`
### HCAccountCashRecord — cash measures (62.5M). PK = (**RecordID**)
`Settled, BankAvailableBalance, TodayIn, TodayOut, TodayDeducted, DailyOpenSettledBalance, AccruedCreditInterest, AccruedDebitInterest, AccruedCustodianFee, LoanBalanceUnderdue/Due/Overdue, LoanBalanceMargin…`.

### HCAccountLocationInstrument — daily stock balance per custody (230M). PK = (**ValueDate**, **AccountID**, **InstrumentID**, **MarketID**, **LocationID**) + `RecordID`
### HCAccountLocationInstrumentRecord — stock measures (22.4M). PK = (**RecordID**)
`Underdue/DueBuy, Underdue/DueSell, NextDayDue…, Today…, HoldForContract, HoldForSettlement, HoldForDeposit, HoldForWithdrawal, HoldForDistribution, HoldForSubscription`.

> To get a balance: join `HCAccountCash` ↔ `HCAccountCashRecord` (and the LocationInstrument pair) on `RecordID`.

### HCAccountMarginSummary (76M), HCAccountMarketCash (20M) — margin & per-market cash aggregates.

### HCBusinessDateToSystemTime — **core business date** (1.4K). PK = (**BusinessDate**)
`BusinessDate, StartTime, EndTime`. Current day = `TOP(1) WHERE EndTime IS NULL ORDER BY StartTime DESC`.

---

## Accruals (BC*)

### BCAccountAccruedInterest — daily accrued interest (76.7M). PK = (**AccountID**, **RegionID**, **CurrencyID**, **ValueDate**)
`DueBuy, DueSell, Settled, LoanBalance, MarginPositionBalance, CreditInterestAmount, DebitInterestAmount, CreditInterestPrincipalAmount, DebitInterestPrincipalAmount, OverrideCredit/DebitInterestAmount, PostingID, PostingDate`.
### BCAccountAccruedCustodianFee (72.8M) — accrued custodian fee, same shape.

---

## Corporate action (MSB*)

### MSBEntitlement — corporate-action master (76.6K). PK = (**EntitlementID**, **State**)
`MarketID, InstrumentID, BalanceType, BalanceDate, ExDate, ElectionPeriodFrom/To, SubscriptionPeriodFrom/To, SubscriptionPrice, RefundDate, AutoExerciseFlag, BookClosedFlag, ApprovedFlag, GenerateVSDInstructionFlag`. Decomposes into `TSBAccountEntitlement` per account via `EntitlementID`.

---

## Gateway (SX*)

### SXGatewayRequestLog — gateway request log (38.3M). PK = (**SourceConnectionUrl**, **RequestID**, **State**)
`MessageBody (nvarchar max), MessageType, LinkupType, LinkupID, AccountID, Destination, NewGatewaySendStatus, BusinessDate, GatewaySequenceNumber, CancelStatus, RelatedRequestID`. Pairs with `SXSynchronizerIncoming/OutgoingMessage`, `SXSynchronizerState` for VSD/bank messaging.

---

## query-patterns — full portfolio balance SQL

Canonical hold/release → balance math (`portQueryString`, BondOMS/Carbon-OMS `handler-common.go`). Settled balance from view `VSBBAccountInstrument`, FULL OUTER JOIN pending deltas from `TSBAccountInstrumentMovement`:

```sql
SELECT COALESCE(b.MarketID, tx.MarketID) AS MarketID,
       COALESCE(b.InstrumentID, tx.InstrumentID) AS InstrumentID,
       b.AvailableBalance, b.WithdrawableBalance
FROM (SELECT * FROM VSBBAccountInstrument b WHERE b.AccountID = N'<ACCOUNT>') b
FULL OUTER JOIN (
    SELECT tx.AccountID, tx.MarketID, tx.InstrumentID,
      SUM(
        CASE
          WHEN tx.MovementType='D' AND tx.HoldType IS NULL THEN +1
          WHEN tx.MovementType='W' THEN -1
          WHEN tx.MovementType='T' THEN -1
          WHEN tx.MovementType='D' AND tx.HoldType IS NOT NULL AND tx.AutoReleaseDate IS NOT NULL THEN +1
          WHEN tx.MovementType='H' AND tx.HoldType IS NOT NULL AND tx.AutoReleaseDate IS NULL AND tx.State='PI' THEN -1
          WHEN tx.MovementType='H' AND tx.HoldType IS NOT NULL AND tx.AutoReleaseDate IS NOT NULL THEN +1
          ELSE 0
        END
        * CASE WHEN tx.State='A' THEN -1 ELSE +1 END
        * CASE WHEN tx.HoldType IS NOT NULL AND tx.AutoReleaseDate IS NOT NULL AND tx.AutoReleaseQuantity IS NOT NULL
               THEN tx.AutoReleaseQuantity
               ELSE tx.Quantity - COALESCE(tx.ReleasedQuantity,0) END
      ) AS PendingQuantity
    FROM TSBAccountInstrumentMovement tx
    INNER JOIN TSBAccountInstrumentMovement pendingtx
       ON pendingtx.MovementID = tx.MovementID AND pendingtx.AccountID = tx.AccountID
      AND pendingtx.MarketID = tx.MarketID AND pendingtx.InstrumentID = tx.InstrumentID
      AND pendingtx.State IN ('PI','PU','PD')
    WHERE tx.AccountID = N'<ACCOUNT>' AND tx.State IN ('PI','PU','A')
    GROUP BY tx.AccountID, tx.MarketID, tx.InstrumentID
) tx ON tx.AccountID=b.AccountID AND tx.MarketID=b.MarketID AND tx.InstrumentID=b.InstrumentID
WHERE (COALESCE(b.Settled,0)<>0 OR COALESCE(b.AvailableBalance,0)<>0 OR COALESCE(b.WithdrawableBalance,0)<>0)
```

Add `AND b.InstrumentID = '<STOCK>'` for a single instrument (`portStockQueryString`).
