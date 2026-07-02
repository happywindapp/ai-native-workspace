# HSC Trading Domain Terminology

Vietnamese securities trading vocabulary that recurs in field names, function names, and user requests. Knowing these prevents misreading domain-specific code as generic. Do not "clean up" struct fields that mirror the XML API wire format.

## Asset classes
- **Equity** — stocks / bonds / OTC (cash account). All `/equity/*` routes.
- **Derivatives** — futures contracts (margin account). All `/derivatives/*` routes.
- **D-Trade** (day trade) — T+0 settlement (sell same day). Toggled per account/ticker via `T1_TRADE_FLAG` and the `equity_dtrades` MongoDB collection.

## Order lifecycle
- **Order Confirmation** — order placed but the client must approve before the broker submits. Approval requires OTP. Configurable max-age before warn/overdue: `MAX_DAYS_MUST_CONFIRM` (default 5).
- **Order Information** — informational/reference orders (also need OTP-approval).
- **ORN** — Order Reference Number, returned by the XML API on order placement.
- **Oddlot order** — fractional / non-standard lot quantity.

## Holdings quantities (named in API responses)
- **Sellable** — available to sell now.
- **Unavailable** — held for settlement / margin.
- **Pending bonus** — dividend/bonus shares not yet credited.
- **Hold for settlement** — T+2 in flight.
- **Intraday bought/sold** — same-day transactions (T+0 capable).

## Margin & collateral
- **ParValue** — nominal / face value of a security (per G3SB).
- **InitialMarginRatio** — % of position value the client must put up.
- **CollateralRatio** — % of security value accepted as collateral for loans.
- **Overriding margin** — per-account exception to default ratios.
- **Buying power** — computed by the XML API (`XML_BUYING_POWER_REQ`), not by the gateway.

## P&L
- **Today change** — intraday P&L (`func-today-change.go`). Cached briefly (~5s) since prices move.
- **Realized** — closed-position P&L. **Unrealized** — open-position mark-to-market.
- **Reference price** — prior close; baseline for margin calc.

## Entitlements
- **ORS** — Oversubscription Rights — buy additional shares (often at IPO / rights issue).
- **Warrant exercise** — convert a warrant to its underlying.

## Insider / risk
- **Insider person** — restricted trader (e.g. company exec). Loaded daily from the RiskControl DB. Rule types: daily limit, monthly limit, with a `Good-Till` expiration date.

## Banking
- **BG / BGW** — Bank Gateway. Lets clients move cash between a bank account and a trading account. `Authorize` = first-time setup, `Reauthorize` = re-confirm.

## Reports
- **g3bs** folder — equity statement files (PDFs etc.). Env `REPORT_FOLDER_EQUITY`.
- **g3bf** folder — futures statement files. Env `REPORT_FOLDER_FUTURE`.
- Both served from PVC-mounted storage.