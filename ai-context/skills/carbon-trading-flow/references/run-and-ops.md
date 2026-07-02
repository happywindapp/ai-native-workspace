# Carbon Run & Ops

Running the Carbon stack locally, applying OMS migrations, and UAT endpoints.

## Local run

When asked to "run local" / "run 3 sources" / "start OMS/Middleware/Terminal" — apply `c:\_project_cabon\run_local\`, do not re-review source.

**Folder `c:\_project_cabon\run_local\`:**
- `README.md` — port map + env wiring (Terminal :2000 → MW :8820/api → OMS :3000 → PG 31902 + Redis 30432), external dependencies, verify commands.
- `patches/carbon-oms-g3sb-nil-rows.patch` — **mandatory, apply before the first OMS build.** Fixes a nil-rows panic at 6 sites in `handler-core-api.go` + `handler-api.go` when the G3SB MSSQL DB lacks a table reference. Pattern: `defer rows.Close()` placed before the `if err != nil { return }` check → nil panic. Fix is a short-circuit return. Covers `/equity/{info,cash,portfolio}/{accountId}` endpoints used by MW `/api/accounts/{accId}/info/check`.
- `scripts/` — 3 files: `start-all.ps1` (builds OMS → starts 3 services, idempotent, skips a service whose port already listens), `stop-all.ps1` (kills by port), `verify.ps1` (health-check chain).
- `envs/{carbon-oms,carbon-middleware,carbon-terminal}.env.local` — standard local env snapshots; copy from here when re-bootstrapping.

### Quick procedure
1. Read `run_local/README.md`.
2. Verify each service's `.env` has the linking vars from the README "Env wiring" (PORT, URL_API_ACCOUNT_SERVICE, NEXT_PUBLIC_API_URL, PG_*, Redis_Connection). Patch `.env` directly if missing; keep secrets (VCB keys, G3SB/XmlApi passwords) unchanged.
3. Check the patch is applied: grep `Could not execute query to get fee master` in `Carbon-OMS/handler-core-api.go` — if the next line has no `return`, apply the patch.
4. Run `powershell -ExecutionPolicy Bypass -File run_local\scripts\start-all.ps1`.
5. Run `verify.ps1` — all 3 endpoints must return 200.

### Traps
- External tunnels the user must open (MoTTY SSH): port 31902 (Postgres), 30432 (Redis). The script only warns; it cannot open them.
- Carbon-Terminal `.env` defaults to `NEXT_PUBLIC_API_URL=https://carbonterminal-uat.hsc.com.vn/api/middleware` + `NEXT_PUBLIC_ENVIRONMENT=uat` → blocks local run; patch to `http://localhost:8820/api` + `prod`.
- FE→MW link: `NEXT_PUBLIC_API_URL=http://localhost:8820/api` (NO `/middleware` — MW mounts routes at `/api`).
- Carbon-Middleware: `npm run dev` (nodemon) — editing `.env` requires a manual kill+restart. Carbon-Terminal: `npm run dev` (Next.js) — `NEXT_PUBLIC_*` is baked at build/start time, full restart needed on env change.
- `.env` files are blocked from Read/Grep by the `privacy-block.cjs` hook — ask the user to verify/edit manually.

## Carbon-OMS migrations — manual apply

Carbon-OMS does **NOT** run `migrate.Up()` at boot. `conn-database.go` only does `sql.Open` + `Ping`. Migration files: `Carbon-OMS/migrations/*.sql` (golang-migrate `NNNNNN_description.{up,down}.sql`).

**Consequence of forgetting:** code deployed before its migration → INSERT/SELECT of a new column fails with `pq: column "X" of relation "..." does not exist`. User-facing symptom: MW returns `bad_request` / `createOrderFail` (OMS returns plain-text 400 from `fiber.NewError`).

**Carbon-Middleware** has no migrations folder and no auto-run — schema is managed via Sequelize `*.model.js`.

### Apply manually
```powershell
$env:PGPASSWORD="<password>"
psql -h localhost -p 31902 -U postgres -d carbon_oms -c "SELECT version, dirty FROM schema_migrations;"
psql -h localhost -p 31902 -U postgres -d carbon_oms -f c:\_project_cabon\Carbon-OMS\migrations\NNNNNN_xxx.up.sql
# or: migrate -path Carbon-OMS/migrations -database "postgres://..." up
```

**Case study — migration 000017** (commit `8f60a98`, 2026-05-07): added column `product_type` to table `order` with a CHECK constraint (`quota`|`credit`). UAT hit `createOrderFail` because the DBA had not applied the migration after the dev→uat merge.

When the user reports `createOrderFail` / `queryDataFail` / `scanClientInfoFail` after a release: check `Carbon-OMS/migrations/` for new files, ask the user to check `SELECT * FROM schema_migrations` on the target DB, propose `migrate up` or `psql -f` if DB version < latest.

## UAT endpoints

| Service | URL |
|---|---|
| Carbon-Terminal (FE) | `https://carbonterminal-uat.hsc.com.vn/home` |
| Carbon-Middleware (API gateway) | `https://carbonterminal-uat.hsc.com.vn/api/middleware` |
| Carbon-OMS (internal cluster DNS) | `http://carbon-oms-svc:3000` |

- Use the Terminal URL for "UAT" / real testing.
- Use the Middleware URL (path `/api/middleware`) to call the API through the gateway from outside.
- `carbon-oms-svc:3000` resolves only inside the cluster (pod-to-pod) — not reachable from a dev machine; port-forward or go via Middleware.