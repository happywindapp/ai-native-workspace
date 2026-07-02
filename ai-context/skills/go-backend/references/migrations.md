# Migrations — Carbon-OMS vs HSC_STP

The two services migrate differently. Get this wrong and a deploy ships code that expects a column the DB does not have.

## Carbon-OMS — manual `.sql`, NOT auto-run

- Location: `Carbon-OMS/migrations/`
- Naming: `NNNNNN_description.up.sql` + `NNNNNN_description.down.sql` (6-digit zero-padded, e.g. `000017_alter_table_order_add_product_type.up.sql`)
- **OMS has no auto-migrate on boot.** Adding a migration file does nothing by itself.

### Adding a migration
1. Create the next `NNNNNN_*.up.sql` and matching `.down.sql`. Keep them simple schema changes (no seed data).
2. The `.up.sql` must be idempotent-safe where possible (`ADD COLUMN IF NOT EXISTS` etc.).

```sql
-- 000018_alter_table_order_add_xxx.up.sql
ALTER TABLE "order" ADD COLUMN "xxx" VARCHAR(20);
-- 000018_alter_table_order_add_xxx.down.sql
ALTER TABLE "order" DROP COLUMN "xxx";
```

### Applying — by hand, every environment
```bash
psql "$PG_CONN" -f migrations/000018_alter_table_order_add_xxx.up.sql
# or, if golang-migrate CLI is installed:
migrate -path migrations -database "$PG_CONN" up
```

**Critical:** when a PR adds an OMS migration, the deploy is incomplete until someone runs it on every env (local, UAT, prod). A skipped migration is a real past incident — `000017` (`product_type` column) missing on UAT caused `createOrderFail`. Always call this out in the PR / handover.

## HSC_STP — `cmd/migrate` (golang-migrate)

- Location: `HSC_STP/migration/sql/`
- Runner: `HSC_STP/cmd/migrate/main.go`, uses `golang-migrate/migrate` v4 via the `HscGoModules/infra/migrate.go` helper.

```go
func main() {
    cfg, err := config.Load(configFile)
    if err != nil { panic(err) }
    mgTool := infra.GetMigrateTool()
    mgTool.CreateDBAndMigrate(cfg.DB, "file://migration/sql")
}
```

### Adding a migration
1. Add `NNNNNN_description.up.sql` + `.down.sql` under `migration/sql/`.
2. Run the migrate binary (separate from `cmd/service` — it does not run inside the API process):

```bash
cd HSC_STP && go run ./cmd/migrate -config-file config.yaml
```

In deployment this is an init-container / pre-deploy step, not part of the service start.

## Rules

- **Always ship a `.down.sql`** alongside every `.up.sql`, both services.
- **Never edit an already-applied migration** — add a new one.
- **OMS:** mention "requires manual migration" explicitly in the PR.
- **STP:** ensure the deploy pipeline runs `cmd/migrate` before `cmd/service`.
- Keep migration SQL portable to the target engine (OMS Postgres; STP Postgres). The G3 MSSQL DB is read-only — never migrate it.

See `carbon-oms-patterns.md` / `hsc-stp-patterns.md` for the surrounding service structure.