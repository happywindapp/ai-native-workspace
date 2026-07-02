# Anti-Patterns to Avoid

Real issues found in the Carbon Go codebase (cross-referenced with the 2026-05-13 architecture review). When **writing** Go here, never introduce these. When **reviewing**, flag them — describe the fix, do not paste exploitable code.

## 🔴 Critical

### A1 — SQL injection via string concatenation
OMS `handler-common.go` builds queries like `... WHERE AccountID = N'` + accountNo + `'`. Any user-influenced value (`accountNo`, `stockName`, codes) is injectable. A regex allowlist (`validAccountID`) is not sufficient.
**Fix:** parameterize — `db.Query("... WHERE account_id = $1", accountNo)` (Postgres) / `@p1` (MSSQL).

### A2 — TLS verification disabled
OMS `main.go` `netClient` uses `tls.Config{InsecureSkipVerify: true}` → MITM on G3/VCB/VTB calls.
**Fix:** `InsecureSkipVerify: false`, `MinVersion: tls.VersionTLS12`; pin internal CA certs if needed.

### A3 — Hardcoded secrets in source
RSA private key literal in OMS `handler-signer.go`; static API keys in STP `.env` committed.
**Fix:** load from env / K8s secret / Vault. Never commit key material. In review, flag the location — don't reproduce the key.

### A4 — External call inside a DB transaction
OMS `handler-api.go` calls `makeCashHold` (G3 SOAP) inside a Postgres `BeginTx`. If PG rolls back, G3 already moved money → ledger drift.
**Fix:** commit local state first, then call the external API; or use an outbox table polled by a worker. External calls live **outside** the tx boundary.

### A5 — Nil-deref on conditional init
OMS registers `POST /equity/vcb/withdraw` unconditionally, but `vcbInstance` is only built when `ENABLE_VCB=true` → nil-deref panic when the flag is off.
**Fix:** register the route only when the dependency exists, or register a stub returning `503` when disabled.

### A6 — Silent error handling
OMS handlers log the error then `return c.Status(200).JSON(emptyRes)` — client sees HTTP 200 with empty data.
**Fix:** return the real status (`500`, `400`) and a structured error body; propagate the error.

## 🟠 High — avoid in new code

### B1 — Global mutable state without a lock
OMS `masterFee map[string]float64` is rewritten by a cron job and read by request handlers → data race.
**Fix:** wrap in a struct with `sync.RWMutex`, or use an atomic snapshot pointer.

### B2 — Panic outside bootstrap
`log.Fatal` / `panic` is fine at init; inside a request handler it takes down the process (only the Fiber/Gin recover middleware saves it).
**Fix:** return errors from handlers; reserve panic for `main`/`Init`.

### B3 — No timeout / retry on external calls
Relying on the global 60s `netClient` only; no per-call context, no retry/backoff.
**Fix:** `http.NewRequestWithContext` with a 10s timeout; retry with backoff on retryable errors (or use `HscGoModules/http`).

### B4 — Regex as input validation
A loose regex (`^[a-zA-Z0-9]{1,}$`) is treated as a security boundary. It is an incomplete allowlist, not SQLi protection.
**Fix:** explicit length/charset validation **and** parameterized queries — defense in depth.

### B5 — String-enum `switch` with `default` error
STP `stp_handler.go` switches on `request.StpType` string values; typos surface only at runtime.
**Fix:** define a typed enum (`type StpType int` + `iota` + `String()`); validate on JSON unmarshal.

### B6 — Mixing raw `database/sql` and GORM in one flow
Adopting `HscGoModules` GORM wrappers into OMS piecemeal can leave one handler half raw-SQL, half GORM — confusing transaction semantics.
**Fix:** migrate a whole flow at once, as a deliberate refactor, or stay consistent with the file's existing style.

## Review output guidance

For each finding: name the anti-pattern (A1–B6), cite `file:line`, state the concrete fix. Do **not** print the injectable query string or the secret value back — reference its location only.

See `go-conventions.md` for the positive checklist, per-service refs for the correct shapes.