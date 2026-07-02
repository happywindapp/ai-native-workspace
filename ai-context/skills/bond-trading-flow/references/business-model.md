# Bond Business Model — BondPlus / TPRL

HSC sells private corporate bonds to investors. **TPRL** (Trái phiếu doanh nghiệp riêng lẻ) is the HNX market; **BondPlus** is the HSC product brand for the same thing — same product, different layer. `MarketFee = "TPRL"`, `DefaultAccountClassId = "BOND_PLUS"`.

## Product

HSC sells bonds it issued or bought from other organizations. Goal: expand EBS, increase ROE. **Professional investors only** (funds, finance companies, enterprises, insurers) — not retail.

### Conditions to trade BondPlus
1. Professional investor — verified by Back Office, outside IBS/ONE.
2. Holds a Base securities account at HSC.
3. Sufficient cash: cash balance ≥ BP contract value, or top up if negative.

### Trading rights by investor type
| Investor type | Buy | Sell |
|---|---|---|
| Ordinary | ❌ | ✅ |
| Professional | ✅ | ✅ |

Ordinary investors may only sell bonds they already own — they cannot buy new from BondPlus. Validate investor type before allowing a Buy action; reject Buy for ordinary investors.

## Account structure

| Account | Description | Format |
|---|---|---|
| Base (Cơ Sở) | ordinary securities account (iTrade) | `011XXX` (no suffix) |
| BP | dedicated TPRL account | `011XXX_BP` |

- One Base → one BP. Cash and bonds are managed separately at the BP level.
- **Account format (10 chars):** `CTCK code(3) + Investor code(3-6) + Account type(4)`, e.g. `011001234_BP`.
- **4th character ∈ `{A,E,B,F,P,C}`:** A = domestic, E = foreign, B = retail (not used for TPRL), F = firm, P = proprietary/tự doanh, C = TBU.
- **Max withdrawal** = `min(CB_Base, 50% × Collateral_BP)`, where `CB_Base` = solicited Base balance, `Collateral_BP` = bond value held as collateral at BP.

## Partners / external systems

| Partner | Role |
|---|---|
| **HNX** | Exchange — order matching, FIX gateway (CBTS), market data (InfoGate) |
| **VSD / VSDC** | Depository — settlement obligation, allocation, account registration; via SWIFT MT over HSC_STP |
| **VCB** (NHTT) | Settlement bank — cash transfer for buy-side payment, MT910 confirm |
| **G3SB** | HSC core system — cash/bond hold/release ledger, contract bookkeeping (SOAP) |
| HSC = **TVLK** | Custodian member (mã thành viên `011`) for accounts in member 011 |

## Transaction types

### 1. Outright (BCGD)
Investor buys/sells bonds → HNX BCGD matching → VSD T0 allocation (MT544/546) → VCB cash settlement (MT910). Full pipeline in `settlement-and-eod.md`.

### 2. Repo (Tái Phát Hành)
Investor transfers bonds as collateral → funder provides cash → at term: unlock collateral + repay principal + interest. Term T+1 → T+365; annual interest % TBU; margin 100%–150%; minimum face value 100M VND. Repo lifecycle uses separate code paths and is largely out of the amend-flow scope below.

## VSD account registration (8-step)

Register/de-register a TPRL trading account at VSD via HSC (TVLK). Professional investor + must already own TPRL.

1. Customer submits dossier (declaration + identity + authorization).
2. DVKH + QLGD verify documents + Base account.
3. NVCK enters in Bond Terminal / BondOMS.
4. QTNVCK approves.
5. NVCK sends MT598 → VSD (`o598`, request = REGISTER).
6. VSD validates → `i598` ACK/NAK.
7. NVCK handles response: ACK → `VSD_REGISTERED`; NAK → `VSD_REGISTRATION_FAILED`.
8. DVKH notifies customer.

**Validation:** account 10 chars + valid 4th char, not duplicate at HSC/VSD; ID 10-digit VN tax ID (foreign longer); 1 account per ID per HSC; Base account ACTIVE in iTrade/G3SB; KYC pass; professional investor confirmed.

**Registration error scenarios:**
1. NAK Duplicate → `DUPLICATE`, manual delete/reuse.
2. NAK Invalid Format (4th char fail) → rollback + retry.
3. Timeout — no `i598` after 24h → `VSD_REGISTRATION_TIMEOUT`, Resend button.

## Fees

- **Brokerage Outright:** `quantity × price × rate`.
- PIT 10% (if applicable), CIT on revenue, handling fee by cash amount.
- **Master fee:** comes from G3SB core, synced daily.
- Buy hold math includes fee but **not tax** — `getTotalCashHold = price × qty × (1 + fee)`.

## Citations

`brd-bondplus-v1.md` §3; `quy-trinh-dang-ky.md` §1/§2/§5; `quy-trinh-thanh-toan.md`.