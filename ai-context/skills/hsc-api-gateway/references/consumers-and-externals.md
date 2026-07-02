# Consumer Apps & External System Integrations

## Consumer applications

The xlsx spec (`docs/Core Api Gateway APIs.xlsx`) tags every endpoint with its consuming app(s). Knowing the consumer narrows which `handler-api-*` path is involved.

| App | What it is | Typical endpoints |
|---|---|---|
| **ONE** | HSC client-facing app (retail trader UI) | `/equity` trading info, `/equity/order-confirmation/*` (client approves own eOrders) |
| **IBS** | Internal Broker System (broker/admin web) | `/equity/order-information/*` (broker reviews/approves client eOrders), `/equity/account` |
| **CSP** | Customer Service Portal (back-office) | Shares some endpoints with IBS (e.g. `/equity/account`) |
| **PnL** | Standalone PnL service (deployed via `PnL-Deployment/`) | Calls `/equity/plport` ("autoport") for portfolio snapshots |

**Owner / maintainer:** All gateway endpoints in the xlsx are credited to **Khoa.NA** (developer + maintainer, snapshot 27/03/2026). The domain expert for this gateway.

## eOrder workflow

**eOrder** = electronic order placed by a client that needs broker approval before submission to the exchange. Two-side workflow:

1. Client places via `/equity/order/new` → status pending.
2. Client sees pending list via `/equity/order-confirmation/list` (**ONE**).
3. Client approves via `/equity/order-confirmation/send-approve` — **requires OTP**.
4. Broker views via `/equity/order-information/list` (**IBS**) and can approve via `/equity/order-information/send-approve` — no OTP (broker is authenticated).

When a user says "ONE shows wrong data" vs "IBS shows wrong data", that narrows the path. New endpoints: ask which app(s) consume it — the answer dictates `order-confirmation` (client) vs `order-information` (admin) placement and auth/OTP requirements.

## External systems integrated by CoreApiGateway

CoreApiGateway is a fan-out gateway. Error codes are prefixed by source system — the prefix tells you which integration failed.

| Nickname | What it is | Protocol | Env var prefix | Notes |
|---|---|---|---|---|
| **G2 / XML API** | Legacy trading core (orders, account info, buying power, statements, ORS/entitlements) | XML/SOAP over HTTP POST | `XmlApi_*` | Session-based, ~11h validity. Session cached in Redis prefix `XmlApi_Session_Key` (default `core`). Refreshed via daily cron + pub/sub `Refresh-xml-session-channel`. Responses regex-parsed. Errors `G2API-*`. |
| **G3SB (DB)** | Equity core SQL Server — stock master, margin %, par values, ISIN/SEDOL/CUSIP, broker, client classes | MSSQL `ApplicationIntent=ReadOnly` | `G3SB_Connection_*` | **Read-only.** Stock master cached in-memory via `atomic.Pointer`, reloaded daily at 7 AM cron. |
| **G3SB (SOAP API)** | Equity write ops — cash/instrument hold/release/deposit/withdrawal | SOAP/XML, `<web:messageTransfer>` envelope, ns `webservice.g3sb.afe.com` | `G3SBApi_Url` / `_Username` / `_Password` | Same backend as G3SB DB, different door. Requests: `CreateCashHold`/`CreateCashRelease` (HOLDTYPE=D), `CreateInstrumentHold`/`CreateInstrumentRelease` (HOLDTYPE T/B/7). Wrappers in `func-cash.go` (`makeEquityCashHold/Release`), `func-instrument.go`. Errors `G3SBAPI-*`. |
| **G3FB (DB)** | Derivatives (futures) core SQL Server | MSSQL read-only | `G3FB_Connection_*` | Mirrors G3SB DB for futures. |
| **G3FB (SOAP API)** | Derivatives write ops — cash hold/release/deposit/withdrawal (no instrument ops) | SOAP/XML, ns `webservice.g3fb.afe.com` | `G3FBApi_Url` / `_Username` / `_Password` | Same SOAP shape as G3SB API. Wrappers `makeDerivativesCashHold/Release` in `func-cash.go`. Errors `G3FBAPI-*`. **No instrument hold/release** — futures don't need it. |
| **RiskControl** | Risk management SQL Server — insider person restrictions, position limits | MSSQL | `RiskControl_Connection_*` | Insider list cached daily. Rule types: daily limit, monthly limit. |
| **DWCSP** | Data warehouse MongoDB — historical orders, trades, balances, monthly reports, derivative closed positions | MongoDB | `DWCSP_Uri`, `DWCSP_DB_NAME` | All `/equity/history/*` and `/derivatives/history/*` query here. |
| **Investment Performance** | Separate MongoDB with auth — performance analytics | MongoDB | `INVESTMENT_PERFORMANCE_Uri`, `_DB_NAME` | Powers `/equity/investment-performance` endpoints. |
| **Trading Core Redis** | Cache + sessions + distributed locks | Redis | `Redis_Connection`, `Redis_Password` | Holds XML session, holiday cache, OTP data (`smsotp-{UID}`, TTL 180s), correlation IDs, snapshot time windows. |
| **Event Store Redis** | Separate Redis for event streams | Redis | `Redis_EventStore_Connection`, `_password` | Distinct from trading core Redis. |
| **Snapshot Market API** | Real-time stock prices (last, prior close) | REST JSON | `MARKET_PRICE_API_URL` | E.g. `https://marketapi-internal.hsc.com.vn`. Used in portfolio P&L, order validation. |
| **Bank Gateway (BG/BGW)** | Banking integration — balance, authorize, reauthorize | SOAP/XML over HTTP | `BGW_ACCOUNT_*` (`CORE_API`, `LOGIN_ID`, `PASSKEY`, `HANDLER`, `CHANNEL`) | Used for `/equity/bg-account/*`. Errors `BGAPI-*`. |
| **Fee Service** | Dynamic fee calculation microservice | (separate) | — | Referenced via `FEESERVICE-*` error prefix; main service mostly precomputes fees. |

### Notes
- **Error-code prefixes:** `G2API-*`, `G3SBAPI-*`, `G3FBAPI-*`, `G3DB-*`, `BGAPI-*`, `OTP-*`, `FEESERVICE-*` — the prefix identifies which integration failed.
- When a user says "G3" unqualified, clarify **G3SB (equity)** vs **G3FB (derivatives)** — separate DBs on separate servers.
- **"Snapshot" ≠ DB snapshot.** It means a market price snapshot from the Snapshot Market API. Equity and derivatives have separate snapshot time windows: `EQUITY_TIME_LOAD_SNAPSHOT`, `DERIVATIVES_TIME_LOAD_SNAPSHOT`.