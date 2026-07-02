# Carbon Monorepo Map

Code-navigation map for the Carbon cluster. Root: `c:\_project_cabon\`.

> **As-of note:** layout consolidated 2026-05. File LOC numbers and line numbers drift — grep a symbol to confirm. Carbon-OMS still carries the legacy package name `OMSPrivateBondApi` (see `carbon-vs-bond.md` rebrand debt).

## The 6 components

| Component | Stack | Port | Role |
|---|---|---|---|
| `Carbon-Terminal/` | Next.js + TypeScript + Redux Saga + MSAL (Azure AD) | 2000 | Web UI for TMD/CS/BO/Account |
| `Carbon-Middleware/` | Node.js + Express + Sequelize/Postgres | 8820 (`/api`) | Gateway: auth, investor mgmt, order receive+validate→forward, STP trigger |
| `Carbon-OMS/` | Go 1.18 + Fiber v2 + Postgres + MSSQL G3SB (read) + Redis + crontab | 3000 | Order management, fork of BondOMS; hold/release G3SB; VietinBank |
| `HSC_STP/` | Go 1.19 + Gin + GORM + zap; Postgres + MSSQL G3SB + Redis | 3006 | VSDC STP — read/write `.fin` MT messages; **shared with Bond** |
| `HSC_STP_ADMIN/` | Next.js; MySQL (users table only) | — | Admin UI for STP operators |
| `HscGoModules/` | Go 1.24 shared library | — | cache/concurrency/crypto/http/infra/middleware — imported by Carbon-OMS + HSC_STP |

Plus `Carbon-Deployment/` (k8s GitOps, README placeholder), `Postman/`, `docs/`, `documents/` (business/regulatory specs), `plans/`, `run_local/`.

## Cross-service flow

```
Browser → Carbon-Terminal → Kong API Gateway → Carbon-Middleware (gateway)
                                                  ├→ Carbon-OMS    (order/cash/balance)
                                                  ├→ HSC_STP       (VSDC MT)
                                                  ├→ HNX InfoGate  (FIX CBS/CBB)
                                                  └→ External (VietinBank, G3, mail, Info Service)
Admin Browser → HSC_STP_ADMIN → HSC_STP
```

MW service files: `handleResFromInfo|Krx|OMS|STP.js`.

## Carbon-Middleware layout

Entry `services.js` (`app.listen()` — no migration auto-run, no `sequelize.sync()` in normal path). Structure:
```
src/{config,constants,functions,middleware,validations}
src/controllers/   account, bond, command, info, login, logs, memeber [sic typo], update-account, user
src/models/        account, addDataMembers, addFieldTableCommand, carbonDeposit, checkConnect, command, log, member, users
src/route/         mirror of controllers
src/services/      account, command, commandHistory, handleResFromInfo|Krx|OMS|STP, log
```
Leftover BondOMS: `bond.js` controller/route + mount `/bonds`; typo file `memeber.js`. No `tests/` folder. `handleResFromSTP.js` ~1253 LOC, `controllers/command.js` ~758 LOC (large files).

## Carbon-OMS layout

Entry `main.go` + `router.go`. Package `OMSPrivateBondApi` (not renamed). Fiber v2 + Postgres + MSSQL G3 + Redis + crontab.
```
main.go, router.go
handler-{api,common,core-api,signer}.go
conn-database.go, retryQueryResultVCB.go, util.go
datastruct/   bank, common, constant, core-api, g3b-api
infra/http/   vcb/ (legacy), mail/   — needs vtb/
migrations/   17+ SQL (init → product_type 000017)
key/, util/, docs/ (swagger), Dockerfile + azure-pipelines.yml
```
`handler-api.go` ~2656 LOC (god-file: handlers + SQL + VCB + asset hold/release + mail).

### Carbon-OMS env vars

- **G3 Core:** `G3SBApi_Url` (SOAP), `G3SBApi_Username`, `G3SBApi_Password`; `REGEX_VALIDATE_BP_ACCOUNT` (default `^[a-zA-Z0-9]{1,}$`).
- **Legacy VCB (must replace):** `VCB_HSC_PRIVATE_KEY`, `VCB_HSC_PUBLIC_KEY`, `VCB_PUBLIC_KEY`.
- **VTB (to add):** `VTB_BASE_URL`, `VTB_CLIENT_ID`, `VTB_CLIENT_SECRET`, `VTB_PROVIDER_ID`, `VTB_BIC_CODE`, `VTB_HSC_PRIVATE_KEY/PUBLIC_KEY`, `VTB_PUBLIC_KEY`, `VTB_KEY_CHECKSUM`.

### Carbon-OMS global state (`main.go`)
`g3sbApiUrl`, `rdb *redis.Client`, `memcache *cache.Cache`, `dbpg *sql.DB` (Postgres), `dbG3SB *sql.DB` (MSSQL G3), `validAccountID *regexp.Regexp`, `validate *validator.Validate`, `vcbInstance *vcb.VCB` (→ `vtbInstance`), `mailerSend *mail.Mail`, `masterFee map[string]float64`.

### G3 Cash Hold pattern (`handler-core-api.go:makeCashHold`)
SOAP POST to `G3SBApi_Url`, `Content-Type: text/xml`, `SOAPAction: urn:messageTransfer`. Body `<REQUEST Type="CreateCashHold" ...>`. Transaction ref `yyyyMMddHHmmss{randSeq(10)}`. Amount `fmt.Sprintf("%.0f", amount)` (integer VND). HoldType `D`. HTTP client `netClient`: 60s timeouts, `InsecureSkipVerify: true`.

## HSC_STP layout

```
cmd/{migrate,service}
handler/   admin, auth, common, configure, stp_allocation, stp_handler, stp_payment_obligation, trade
internal/
  common/error.go
  domain/hsc_stp/{constant,model,repo,usecase}
    usecase/  admin, itrade, ismart_notify, send_stp_account/instrument/rights_allocation, stp,
              stp_allocation, stp_confirm_payment_obligation, stp_list, stp_register_account,
              stp_related_compare, stp_send_noti, stp_store, stp_update_account, template, trade
  infra/{http/itrade, http/smartNotify, repo}
    repo/    i598, o518/o544/o546/o548/o564/o567/o568/o598 + account_info, activity_logs,
             base_sql, send_stp, repo_csv, trade_statement_csv
migration/sql/, apidocs/, server/, config/
```
MT coverage: 508/518/544/546/548/564/567/568/598 (in/out per `i*`/`o*` prefix).

## Files by feature

| Feature | Files |
|---|---|
| Login | MW `controllers/login.js`; Terminal `pages/login.tsx`, `redux/sagas/` |
| Place order | Terminal → MW `controllers/command.js` + `services/command.js` → OMS `handler-api.go` |
| Register account | MW `controllers/account.js` → STP `usecase/stp_register_account.go` |
| MT generation | STP `internal/infra/repo/stps_o*.go` |
| Cash hold | OMS `handler-core-api.go:makeCashHold` |
| Custody/match | STP `usecase/stp_allocation.go`, `stp_related_compare.go` |
| Smart notify | STP `usecase/ismart_notify.go`, `stp_send_noti.go`, `infra/http/smartNotify/` |
| Info Service | MW `services/handleResFromInfo.js`, `controllers/bond.js`, `route/bond.js` |
| VCB/VTB | OMS `handler-signer.go`, `infra/http/vcb/`, `retryQueryResultVCB.go` |
| DB / migrations | OMS `conn-database.go`, `migrations/` |

## System architecture facts

- **API GW:** Kong (rate limit, routing, logging).
- **Auth:** external = Microsoft Entra ID JWT bearer (Terminal↔Middleware); internal = K8s network policy (Middleware↔OMS/STP no auth).
- **DB engines (mismatch):** Carbon-Middleware Postgres · Carbon-OMS Postgres + MSSQL G3SB(read) · HSC_STP Postgres + MSSQL G3SB + Redis · HSC_STP_ADMIN MySQL.
- **User roles:** Admin (full) · CS (register investor, manage info, view) · TMD (place orders, order book, STP creation, portfolio) · Account (view orders/settlement, fund-deduction monitor, reconciliation).
- **Error codes:** prefix `ERR_XXXX` — 1001-1003 Auth · 4001-4008 Order · 5001-5004 Investor · 6001-6003 STP · 7001-7003 Fund deduction · 9001-9005 Internal.
- **Gitflow:** main (prod) · dev · uat (tagged `vX.X.X-uat`) · RC-vX.X.X · feature/* · hotfix.
- **Logging:** dev/uat DEBUG, prod INFO. Mask investor PII and bank account numbers to last 4; never log tokens/passwords; order & STP content allowed (audit).

## Known architecture risks (arch review 2026-05-13 — stable, structural)

These are *standing* risks, not dated incidents — verify against current code before acting.

1. **Service-to-service auth = static string committed.** MW `myAPIKey/bondKey = "carbon-terminal"`; STP `API_KEY_TERMINAL`; MW JWT secret committed; OMS embeds an RSA private key in source.
2. **STP `/admin/*` auth commented out** in `server/init.go`; STP_ADMIN calls STP directly from the browser → admin console exposed on LAN.
3. **MW route protection inverted.** `accountRoute.use(middlewareLogin)` is placed *after* the route registrations → several endpoints (`/bonds`, `/info`, `/logs`, `/members`) unauthenticated.
4. **OMS runs G3 SOAP + VCB transfer inside a Postgres transaction** (`makeG3Order`, `callTransferToVCB` within `BeginTx`) → PG rollback does not roll back external calls → ledger drift.
5. **SQL injection in OMS** `handler-common.go` / `handler-core-api.go` — `fmt.Sprintf` of JSON body fields into MSSQL G3SB; only mitigation is an env-overridable regex.
- Additional: TLS disabled in several places (MW `NODE_TLS_REJECT_UNAUTHORIZED=0`, OMS `InsecureSkipVerify`, PG `sslmode=disable`); STP lane-leak (`stp_list.go` does not filter `from_service`); the 6/9-char rule is not enforced at any layer; no shared OpenAPI/proto types; HscGoModules under-utilized (OMS imports only 2/12 sub-packages).