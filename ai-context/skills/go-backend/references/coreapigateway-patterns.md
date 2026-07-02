# CoreApiGateway Patterns (Fiber v2, REST↔XML/SOAP translation)

`CoreApiGateway/` (in `c:\_core_api_gateway`) — Go 1.23, Fiber v2, ~13.7k LOC, port **3030**. The API gateway fronting HSC (Ho Chi Minh Securities) trading: translates modern REST/JSON to/from a legacy XML/SOAP core ("G2") and aggregates SQL Server / MongoDB / Redis sources.

This skill covers the **Go-level conventions**. For domain navigation (wire codes, consumers, deployment overlays, endpoint catalog) use the `hsc-api-gateway` skill.

## Layout

Flat, not clean-arch. The convention is a 3-stage pipeline per request:

```
handler-api-*.go   → func-*.go        → external systems
(Fiber handler)      (business logic +   (G2 XML, G3SB/G3FB SOAP,
                      external calls)     SQL Server, Mongo, Redis)
```

- Entry `main.go`, routes `router.go`.
- Datastructs are **split by purpose**:
  - `datastruct/api-func.go` — REST DTOs (request/response bodies).
  - `datastruct/core.go` — XML/SOAP types for the legacy G2 core.
  - `datastruct/common.go` — cached domain models.
- `util.go` holds precompiled regex used to parse XML responses.

When adding an endpoint: handler in `handler-api-*.go`, business logic in a `func-*.go`, REST DTO in `api-func.go`, any XML struct in `core.go`. Match the Equity vs Derivatives split — handlers/functions/datastructs are duplicated end-to-end on purpose (`/equity/*` vs `/derivatives/*`).

## Fiber handler pattern

Same Fiber v2 `(c *fiber.Ctx) error` signature as the OMS services. Handler parses + validates the REST DTO, calls into a `func-*.go`, returns JSON. Nearly every write endpoint carries an `autoApprovalFlag` (`Y`/`N`) field — `Y` = auto-approved, `N` = queued for back-office approval. Follow that convention on new write endpoints.

## REST ↔ XML/SOAP translation (the core job)

### Outbound XML — never string-concat user input
There is a custom XML-escape helper (escapes `&<>"'`). All user-supplied strings going into an XML request MUST pass through it. Bypassing it = XML injection. Define the request struct in `datastruct/core.go`; build the body via the helper, not raw `fmt.Sprintf` of user data.

### Inbound XML — parsed by regex, not an XML parser
`util.go` contains precompiled regexes that extract scalar fields from SOAP responses. This is **brittle by design**: any whitespace/element-ordering change in the upstream XML silently breaks a field. A "field came back empty" bug → suspect the regex first. Adding a new extracted field = add a new precompiled regex.

### G2 session lifecycle
Login returns a session ID valid ~11 hours, cached in Redis under `XmlApi_Session_Key` (default `core`). A daily cron (`JOB_TIME_SESSION_G2`) refreshes it; Redis pub/sub channel `Refresh-xml-session-channel` invalidates cached sessions cluster-wide. A midnight `G2API-500` / auth spike usually means the refresh cron failed.

### Amount strings
Many numeric fields arrive as comma-separated strings (`"1,234,567.89"`). Strip commas before parsing — a `string`-typed amount has a fixed format, it is not free-form.

## Error-code prefixes

The prefix tells you which system failed: `G2API-*` (legacy XML), `G3SBAPI-*` (equity SOAP), `G3FBAPI-*` (derivatives SOAP), `G3DB-*` (G3 DB), `BGAPI-*` (Bank Gateway), `OTP-*`, `FEESERVICE-*`. Preserve these when surfacing errors.

## XmlAutoLogin (sibling service)

`XmlAutoLogin/` — a tiny standalone Go cron service (~140 LOC), cron `0 0 5 * * *` (5 AM Asia/Bangkok). Logs into the legacy XML API per endpoint in `JOB_XML_LIST` as a daily keepalive/smoke-test. It does **not** write the gateway's Redis session — the gateway manages its own refresh. Its HTTP client uses `InsecureSkipVerify: true` because internal endpoints use self-signed certs — acceptable on the internal network for this keepalive service ONLY; never copy that pattern into client-facing gateway code.

## Conventions & gotchas

- Timezone `Asia/Bangkok` is hardcoded (= Vietnam time); currency VND only.
- OTP is a two-step flow (`POST /otp/request` → action with `otp`), never inline. Redis key `smsotp-{UID}`.
- **Known field-name typos are part of the wire contract** — `contractRefence` (collateral-loan), `bankAccontNumber` (derivatives account). Do NOT "fix" them; renaming breaks ONE/IBS/CSP callers.
- ORS uses `autoApproval` as `T`/`F`, inconsistent with the `Y`/`N` `autoApprovalFlag` elsewhere — match the endpoint, don't normalize.
- `poc/` mirrors the structure but is NOT deployed — scratch only.
- The xlsx spec + Postman collection are the contract source of truth, not the Go source.

## Anti-patterns (flag in review)

| Issue | Fix |
|---|---|
| Constructing XML by `fmt.Sprintf` of user input | Use the XML-escape helper |
| Renaming a typo'd wire field (`contractRefence` etc.) | Leave it — it's the contract |
| `InsecureSkipVerify: true` in client-facing code | Only `XmlAutoLogin` keepalive may do this; gateway code must verify TLS |
| Hardcoded G2/G3SB/G3FB/Bank credentials or session IDs | Load from env / overlay ConfigMap-Secret |
| "Improving" the regex parser into a full XML parser ad-hoc | Coordinate — every extracted field has a matched regex |

Shared cross-service anti-patterns (A1 SQLi, A3 secrets, A6 silent errors) also apply — see `anti-patterns.md`.

See `hsc-api-gateway` skill for domain navigation, `g3-core-integration` for G3SB/G3FB SOAP transport, `carbon-oms-patterns.md` / `bondoms-patterns.md` for the sibling Fiber services.