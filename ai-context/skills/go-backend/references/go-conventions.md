# Go Conventions In This Repo

Idioms actually used across Carbon-OMS / HSC_STP, plus the writing checklist. Where the two services differ, match the service you are editing.

## Error handling

- **Propagate, don't swallow.** OMS's habit of `log.Printf(...)` then returning an empty 200 is wrong — return the error / a real status.
- **Wrap with `%w`** so callers can `errors.Is` / `errors.As`: `return fmt.Errorf("list orders: %w", err)`. Neither service does this consistently yet — new code should.
- Sentinel errors: STP keeps them in `internal/common/error.go` (`ErrNotFound = errors.New("not found")`).
- Init failures panic (`checkErr`/`log.Fatal` in OMS, `panic(err)` in STP) — acceptable only at bootstrap, never in request handlers.

## Context

- **STP:** propagate `ginwrapper.Context` / `contxt` through handler → usecase → repo; it carries the request ID and logger.
- **OMS:** uses a global `ctx = context.Background()` that is never cancelled. New external calls should still create a real timeout context: `ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second); defer cancel()`.

## Concurrency

- OMS: `crontab` jobs + occasional fire-and-forget `go fn()`. If you spawn a goroutine, make it cancellable and guard any shared state with a mutex.
- STP: cron-based folder polling (15s tick), no ad-hoc goroutines.
- `HscGoModules/worker` (gocraft/work) and `HscGoModules/concurrency` exist for real job queues — prefer them over hand-rolled goroutine pools.

## Struct tags

- `json:"..."` on all DTOs (both services).
- STP also uses `form:"..."` (Gin binding) and `yaml:"..."` (config structs).
- Add `validate:"..."` tags when the handler runs `validator`.

## Logging

- **OMS:** stdlib `log` (`log.Printf`, `log.Println`, `log.Fatal`) — no levels.
- **STP / HscGoModules:** `go.uber.org/zap` sugared — `zap.S().Infof`, `.Errorf`, `.Debugf`; or the `contxt` logger which prefixes the request ID.
- Never log passwords, private keys, tokens, full card/account data.

## Naming & layout

- Go files: `snake_case` or the service's existing pattern (OMS uses `handler-foo.go` with a hyphen — match it inside OMS; STP uses `snake_case.go`).
- Exported identifiers `PascalCase`, unexported `camelCase`.
- OMS DTOs → `datastruct/`; STP DTOs → `handler/model/` or `internal/domain/hsc_stp/model`.

## "Writing Go here" checklist

**Structure**
- [ ] New code goes in the right place: OMS flat `handler-*.go` + `datastruct/`; STP `handler/` + `usecase/` + `repo/`.
- [ ] Reuse `HscGoModules` before writing bespoke infra/cache/http/crypto.
- [ ] Keep handlers thin — push logic to a usecase/service func (especially STP).

**Database**
- [ ] Parameterized queries only (`$1`, `@p1`) — never string-concat input.
- [ ] `defer rows.Close()` right after `Query()`.
- [ ] Return an error + real status on DB failure, not an empty 200.
- [ ] OMS = raw `database/sql`; STP = GORM. Don't mix in one handler.

**Errors & context**
- [ ] Wrap errors with `%w`; propagate up; map to HTTP status at the handler.
- [ ] Use a timeout context for every external call.

**External calls (G3 / VCB / VTB / VSDC / iTrade)**
- [ ] Never inside a DB transaction (see `anti-patterns.md` A4).
- [ ] Timeout + retry/backoff; idempotency key for payment/settlement calls.
- [ ] TLS verification on — no `InsecureSkipVerify: true`.

**Config & secrets**
- [ ] OMS reads `.env`; STP reads YAML with `${ENV}` expansion. No hardcoded keys/secrets.

**Migrations**
- [ ] Ship `.up.sql` + `.down.sql`. OMS = note "apply manually"; STP = ensure `cmd/migrate` runs pre-deploy.

**Build check**
- [ ] After editing, build the service: `cd <service> && go build ./...` (and `go vet ./...`).

See `anti-patterns.md` for what to actively flag, the per-service refs for concrete code shapes.