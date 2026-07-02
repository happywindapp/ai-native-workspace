# Debug Queries — Bond Data Investigation

Reusable SQL for investigating bond data during an incident. BondOMS Postgres `dbpg` owns `order`, `logs`, `log_hnx`.

> `order` is a reserved word in Postgres — always quote it as `"order"`.

## Pull data by date — BondOMS Postgres (`dbpg`)

```sql
-- Logs for a given day (adjust the date range)
SELECT * FROM logs
WHERE created_at >= '2026-04-17 00:00:00'
  AND created_at <  '2026-04-18 00:00:00'
ORDER BY id ASC;

-- Orders for a given day
SELECT * FROM "order"
WHERE created_at >= '2026-04-17 00:00:00'
  AND created_at <  '2026-04-18 00:00:00'
ORDER BY id ASC;
```

## Notes

- `psql` is available via Bash for ad-hoc debugging.
- `dbG3SB` (MSSQL, read-only mirror of G3SB) is queried by BondOMS for account/portfolio/cash/fee data — see `bondoms-map.md` → "G3SB query functions".
- For status enums when reading rows: `hnx_status` / `vsd_status` / `bank_status` decode in `BondOMS/datastruct/constant.go` and `docs/Bond Riêng Lẻ - status.md`.
- Filter TTDT vs BCGD rows with `transaction_type = 'outright_ttdt'` vs `'outright_bcgd'` (legacy rows are NULL).