# G3SB API Surface — What Bond Uses, Where It's Invoked

G3SB = G3 Securities back-office, HSC's core system. For TPRL it holds investor accounts, cash/bond holds, contracts, and custody. Source spec: `g3sb-api.md` v1.09 (N2N-AFE 2019-11-14).

> Scope here is the *call surface* — which operations bond uses and where. Hold/release *business rules* → `bond-trading-flow`.

## Transport

- HTTP/HTTPS POST, SOAP XML.
- Mandatory ZIP + Base64: XML → PKZIP → Base64 → `<Zip>` tag. Signing optional (HMAC-SHA256 or RSA).

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <Operation>
      <Zip>{Base64 ZIP of XML}</Zip>
      <Signature>{optional}</Signature>
    </Operation>
  </soap:Body>
</soap:Envelope>
```

G3SB exposes raw SOAP only — there is no wrapper/SDK on the G3SB side. The wrapper-function pattern lives in BondOMS.

## Core SOAP operations used by bond

| Operation | Purpose |
|---|---|
| Create / Update / Query Account (`GetAccount` / `QueryAccount`) | Investor account lifecycle + lookup |
| `CreateCashDepositTransaction` | Cash hold (lock for purchase) |
| `CreateCashWithdrawalTransaction` | Cash hold on withdrawal (after a bond sale) |
| Create Bond Hold | Lock bond inventory for a sale |
| `ReleaseHold` (cash or bond) | Release a hold — partial release allowed |
| `CreateAccountContract` | Contract terms + fee schedule |
| Query Cash Statement / Query Instrument Statement | Statement / reconciliation reads |

## Where bond invokes G3SB

- **BondOMS** is the only caller. It wraps the raw SOAP ops behind helper/wrapper functions.
- Hold/release wrappers used in the order flow: `makeCashHold` / `makeStockHold` / `makeCashRelease` / `makeStockRelease`; contract/match helpers `makeTransactionBond`, `makeG3Order`.
- TTDT path: `holdAssetOrder` / `releaseAssetOrder` invoked from `handler-quote-place.go` and `handler-quote-amend-cancel.go` (rule: hold(new) before release(old)).
- `dbG3SB` (MSSQL, read-only mirror) is queried directly for account/portfolio/cash/fee data via `handler-common.go` query-string functions — see `bondoms-map.md`.

## Settlement role (where each call sits)

1. HNX match → BondOMS receives MT518 from VSD.
2. BondOMS → G3SB `CreateCashHold` (investor acct, amount, T+0) → HoldRef.
3. Investor confirms payment to VCB.
4. BondOMS → G3SB `ReleaseHold` (HoldRef, full) → cash released.
5. VSD receives MT598 from HSC → allocates securities.
6. Complete: investor +securities/−cash; seller −securities/+cash.

## Key field shapes

- `CreateCashDepositTransaction` / `...Withdrawal`: `TransactionNo` (unique, = VCB transactionNo), `AccountNo`, `Amount` (Decimal VND), `Currency` "VND", `ValueDate` (usually T+0), `Description`.
- `ReleaseHold`: `TransactionNo` (original hold ID), `ReleaseAmount` (partial OK), `Reason`.
- `CreateAccountContract`: `AccountNo`, `ContractType` ("TPRL"/"REPO"/"OUTRGHT"), `EffectiveDate`, `FeeSchedule[]`, `RiskLimit`. Fees are embedded in `CreateContract.FeeSchedule[]` — there is no separate fee-sync op.

## Validation

- Account: `\d{3}[A-Z]{3}_[A-Z]{2}` (e.g. `011ABC_BP`).
- Amount: 2 decimal, `0.01` → `999,999,999.99` VND.
- ISIN: `^[A-Z]{2}[A-Z0-9]{9}[0-9]$`.
- Date: `YYYY-MM-DD` ISO 8601.

## Error codes

| Code | Meaning |
|---|---|
| 00000 | Success |
| 10001 / 10002 | Account not found / inactive-frozen |
| 20001 / 20002 | Insufficient balance / hold exceeds balance |
| 30001 / 30002 | Tx already exists (idempotent reuse) / invalid amount-currency |
| 40001 / 40002 | Contract not found / contract inactive |
| 50001 / 99999 | System error (retry 5min) / unhandled |