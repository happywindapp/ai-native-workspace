# HscGoModules — Shared Library

`HscGoModules/` — shared Go lib, module path `dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git`. Imported by Carbon-OMS and HSC_STP. **STP uses it well; OMS largely ignores it and duplicates functionality** — when adding to OMS, prefer adopting these packages over rolling your own.

## Package catalog

| Package | Provides | OMS uses? | STP uses? |
|---|---|---|---|
| `cache/` | `ICache` interface + Redis impl | ❌ uses `patrickmn/go-cache` | ✅ |
| `concurrency/` | worker-pool / semaphore limit helpers | ❌ | ❌ |
| `contxt/` | `MyContext`, structured logger factory, request ID | ❌ uses `log.Printf` | ✅ heavily |
| `crypto/` | RSA, AES, Triple-DES ECB | ❌ hardcodes RSA in `handler-signer.go` | ❌ |
| `ginwrapper/` | `Context` wrapper over `*gin.Context`, response helpers | ❌ (Fiber) | ✅ all handlers |
| `http/` | `MakeRequest()` with retry + timeout | ❌ manual `netClient` | ⚠️ imported, underused |
| `infra/postgres/` | `InitPostgres()` → `*gorm.DB`, replica config | ❌ raw `sql.Open`+`lib/pq` | ✅ |
| `infra/sqlserver/` | `InitSqlServer()` → `*gorm.DB` | ❌ raw `sql.Open` | ✅ |
| `infra/redis/` | `InitRedis()` factory | ❌ manual `redis.NewClient` | ✅ |
| `infra/migrate.go` | `GetMigrateTool()`, `CreateDBAndMigrate()` | ❌ manual `psql -f` | ✅ in `cmd/migrate` |
| `middleware/` | `SetRequestID()`, `SetupLog()`, `RateLimit()` | ❌ | ✅ |
| `worker/` | job queue via `gocraft/work` + Redis | ❌ uses `crontab` | ❌ uses `cron` |
| `util/` | array/file/formatter/json/math/secure-string helpers | ❌ | ⚠️ minimal |
| `project/` | `datafeed/`, `krx/` — project-specific feeds | ❌ | ❌ |
| `tracer/` | Datadog tracer factory | ❌ | ❌ |

## How STP imports it

```go
import (
    contxt        "dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git/contxt"
    postgres_wrapper "dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git/infra/postgres"
    redis_wrapper "dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git/infra/redis"
    middleware    "dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git/middleware"
    ginwrapper    "dev.azure.com/HSC-Inhouse-Dev/Tools/_git/HscGoModules.git/ginwrapper"
)
```

## Guidance when writing Go here

- **Before writing infra/cache/http/crypto code, check this table.** If a package covers it, import it instead of duplicating.
- **STP** — keep using `contxt`, `infra/*`, `middleware`, `ginwrapper`. New cross-cutting concern → see if `concurrency/`, `worker/`, `tracer/`, `util/` already solve it.
- **OMS** — it imports `HscGoModules` only as an indirect dep today. Adopting a package (e.g. `infra/postgres`, `http.MakeRequest`) is an improvement, but it is a **deliberate refactor** — flag it, don't silently switch a working flow. Do not adopt piecemeal in a way that mixes raw `sql.DB` and GORM in the same handler.
- **Go version:** HscGoModules targets Go 1.24, OMS is 1.18, STP is 1.19. Confirm the consuming service's `go.mod` can build the package version before bumping; a mismatch is an open question, not a given.
- **Editing HscGoModules itself** — it is shared by both services (and possibly others). A change there is cross-cutting: check both consumers build, and treat it like a library release, not a local edit.

See `carbon-oms-patterns.md` / `hsc-stp-patterns.md` for how each service wires these in, `anti-patterns.md` for the duplication debt this table reflects.