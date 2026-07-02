# API Conventions, Gotchas & Legacy G2 XML Quirks

## Cross-cutting API conventions

### `autoApprovalFlag` (`Y` / `N`)
Present on nearly every write endpoint (cash/instrument deposit/withdraw/hold/release, account create/update, interest-setting, fee-setting, ORS create, collateral-loan create/update). `Y` = auto-approved at the gateway/G3 layer; `N` = queued for back-office approval. New write endpoints should follow this convention.

### OTP two-step flow
Sensitive flows do **not** send OTP inline. Instead:
1. `POST /otp/request` with `{uid, expiredTime?, maxResend?, maxFail?}` → returns `requestId`.
2. Call the action (e.g. `/equity/order-confirmation/send-approve`) supplying the `otp` value.
3. Gateway internally re-verifies, or `POST /otp/verify` with `{uid, requestId, otp}` is called explicitly.

OTP defaults: **180s** expiry, max **3** resends, max **3** failures. Stored in Redis under key `smsotp-{UID}`. Do not invent inline OTP verification.

### Pagination
Used in `/equity/order-confirmation/list`, `/equity/order-information/list`, etc.:
```json
{
  "pagination": true,
  "page": 1,
  "pageSize": 50,
  "orderBy": { "orn": "asc", "placementDate": "desc" }
}
```
`pagination: false` (or absent) returns the unpaginated set.

### Bulk-update pattern
Some endpoints accept either a single record or an array:
- `/equity/dtrade/update` — body is a JSON array of `{accountNo, optFlag, webFlag, tglFlag}`.
- `/equity/account/interest-setting` and `fee-setting` — accept top-level fields **or** a `record` array for multi-row updates.

## Gotchas

### Date format inconsistency
Most endpoints use `YYYY-MM-DD` (ISO). **But** `/equity/entitlement/ors/list-subscription` accepts `startDate`/`endDate` as `YYYYMMDD` (no dashes). They are not interchangeable — match the exact format the endpoint expects.

### Known field-name typos — do NOT "fix" them
| Endpoint(s) | Typo'd field | Correct-looking name (do not use) |
|---|---|---|
| `/equity/collateral-loan/create` & `/update` | `contractRefence` | ~~contractReference~~ |
| Derivatives account create | `bankAccontNumber` | ~~bankAccountNumber~~ |

Fixing a typo breaks the wire contract with callers (ONE/IBS/CSP). If a rename is ever truly needed, coordinate with all callers.

### `autoApproval` on ORS is `T`/`F`, not `Y`/`N`
`/equity/entitlement/ors/create` uses `autoApproval` as the strings `T` / `F` — an inconsistency with the `autoApprovalFlag` `Y`/`N` convention everywhere else.

### Endpoint status (per xlsx, snapshot 27/03/2026)
- Most endpoints are `PROD`.
- `/equity/insider-person` is `UAT` only — not promoted to production.
- `/equity/fee-info` and `/derivatives/fee-info` are `Waiting` — declared but not implemented. Confirm with Khoa.NA before promising these to a client integration.

### E-Invoice is a separate service (port 3080)
The Postman collection includes `GET /invoice`, but it is hosted on a **different service at port 3080** (`http://localhost:3080/invoice`), not the main gateway on 3030. It is NOT in `CoreApiGateway/router.go`. The xlsx documents it in the "E-Invoices APIs" sheet for completeness only.

### KRX URLs in Postman
Some Postman entries point at production KRX URL (`https://core-api-gw-krx.hsc.com.vn`) instead of `{{coreAPIHost}}`. These are convenience entries for testing against KRX prod — not architecturally meaningful. The same endpoints exist on local/UAT.

## Legacy G2 XML API quirks

The XML API (a.k.a. "G2") is the legacy trading core. Its integration has several non-obvious behaviors that recur in bug reports:

### Session lifecycle
- Login returns a session ID valid **~11 hours**.
- Session stored in Redis under prefix `XmlApi_Session_Key` (default `core`).
- Daily cron at `JOB_TIME_SESSION_G2` (default `0 0 * * *`) refreshes it.
- Redis pub/sub channel `Refresh-xml-session-channel` invalidates cached sessions cluster-wide.
- **A "G2API-500" or auth-failure spike at midnight usually = session refresh failed.** Check both `CoreApiGateway` cron logs AND `XmlAutoLogin` cron (keepalive at 5 AM).

### `XmlAutoLogin` is not the source of truth
The standalone `XmlAutoLogin` service does a daily login but does NOT push the session into the gateway's Redis. The gateway manages its own session refresh independently. `XmlAutoLogin` is effectively a connectivity smoke-test / keepalive ping per endpoint in `JOB_XML_LIST`.

### Response parsing uses precompiled regex, not an XML parser
`util.go` contains regex patterns to extract fields from SOAP responses. **Brittle:** any whitespace/ordering change in the XML server's output can silently break parsing. A "field came back empty" bug in equity/derivatives flows → suspect the regex first.

### Amount strings
Many numeric fields come back as strings with comma thousands-separators (e.g. `"1,234,567.89"`). The gateway strips commas before parsing. A `string`-typed amount field has a specific format — not free-form.

### XML escaping is manual
There is a custom XML-escape helper for outbound requests (escapes `&<>"'`). Bypassing it = XML injection risk. Never construct XML by string-concatenation outside the helper.

### Multi-endpoint failover is not real
`JOB_XML_LIST` (in `XmlAutoLogin`) is a list of XML server IPs, but the main gateway uses a single `XmlApi_Url`. The list is for warming all endpoints, not runtime failover.

### TLS verification disabled in XmlAutoLogin
`XmlAutoLogin`'s HTTP client uses `InsecureSkipVerify: true` because internal endpoints use self-signed certs. Acceptable on the internal network — do NOT copy this pattern to client-facing code.

## Adding a new XML API request
Follow the existing pattern: define the struct in `datastruct/core.go`, add a precompiled regex if you need to extract scalar fields from a list, and use the XML-escape helper for any user-supplied string.