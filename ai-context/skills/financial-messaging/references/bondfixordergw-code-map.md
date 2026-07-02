# BondFIXOrderGW — Code Map

Navigation map of the **BondFIXOrderGW** service (Go, module `pbordergw`) — the FIX gateway between HSC bond trading and HNX. Use this to locate the FIX/NATS/session code when debugging the FIX layer or building an HNX Simulator.

> Paths below are repo-relative unless marked absolute. Absolute paths assume the standard clone root `c:\_project_git\`; adjust to your workspace. Line numbers drift — grep the named symbol to confirm.

## FIX Lib & Custom Dialect

- **Lib core:** `HnxQuickfix v1.0.14` (Azure DevOps fork of QuickFIX/Go upstream v0.7.0 — `dev.azure.com/HSC-Inhouse-Dev/PrivateBond/_git/HnxQuickfix.git`). Ships custom HNX tags + message types.
- **Local source clone:** `c:\_project_git\HnxQuickfix\` — go 1.20, used to read spec + extend.
- **Spec source of truth:** `c:\_project_git\HnxQuickfix\spec\FIX44.xml` (~6920 lines). Run `make generate` → regenerates `gen/*`.
- **Acceptor API** (for HNX Simulator): `quickfix.NewAcceptor(app, storeFactory, settings, logFactory)`. `app` implements the `Application` interface — 7 methods: `OnCreate` / `OnLogon` / `OnLogout` / `ToAdmin` / `ToApp` / `FromAdmin` / `FromApp`.
- **Custom tags:** `internal/fix/tag/` — integer tags e.g. 6363, 6464, 9735–9745, 4488, 537, 549, 6251, 109, 334.
- **Custom fields:** `internal/fix/field/`
- **Enums:** `internal/fix/enum/`
- **Custom message structs:** `internal/fix/fix44/` — 20+ subdirs:
  - `newordercross/` (35=s) · `crossordercancelrequest/` (35=t) · `crossordercancelreplacerequest/` (35=u)
  - `executionreport/` (35=8) · `securitystatus/` (35=f) · `tradingsessionstatus/` (35=h)
  - Session: `logon/`, `logout/`, `heartbeat/`, `reject/`, `resendrequest/`, `sequencereset/`, `testrequest/`
  - Quote: `nquote/`, `nquoterequest/`, `nquoteresponse/`, `nquotestatusreport/`, `nquotestatusreportfirm/`, `quotestatusreport/`
  - Multileg: `nmultilegorder/`, `nmultilegordercancelrequest/`, `nmultilegorderreplacerequest/`, `reportnmultilegorder/`, `execorderrepos/`, `multilegordercancelreplace/` (file `MultilegOrderCancelReplace.generated.go`)
- **Higher-level msg structs:** `internal/msgstruct/` — Go structs wrapping fix44 types, used in the client.

## Session Setup

- **Config:** `config/fixconfig.cfg` (QuickFIX format) · **App config:** `config/config.yaml`
- **Mode:** Initiator
- **Production target:** `SocketConnectHost=192.168.212.196:1369`
- **Identities:** `SenderCompID=011.05TESTGW`, `TargetCompID=HNX` (config) — actual logs show sender `011.06GWTEST`.
- **Auth:** Logon (35=A) sends `553=username 554=password`, e.g. `553=011.06GWTEST 554=<password>`. Credentials live in `config/fixconfig.cfg` — never hardcode or echo the real value.
- **HeartBtInt:** 30s · **Time zone:** Asia/Bangkok, `UseLocalTime=Y`

## NATS Publish — Single Entry Point

- **File:** `internal/client/write_obj_log.go` → `sendMsgToNats(appClient *Client, msg *quickfix.Message) error`
- Every inbound FIX (`FromApp` / `FromAdmin`) publishes through this function → OMS consumes it to drive order status. Outbound FIX also publishes (for audit/replay).
- Metrics: `metrics.Global().IncNATSPublish()` / `IncNATSPublishError()`

## Inbound / Outbound Handlers

- `internal/client/in_msg_hdl.go` — inbound message routing
- `internal/client/out_msg_hdl.go` — outbound message dispatch
- `internal/client/main.go`:
  - `FromAdmin` — Logon / Logout / Heartbeat handlers
  - `FromApp` — application messages
  - `Client` struct — holds `MessageRouter`, `hnxSession *quickfix.SessionID`, `Nats StreamPublisher`, `Usecase`, `logger`
  - Session-safe accessors: `GetSession()` / `setSession()` guarded by `sync.RWMutex`

## Store

- **File store:** `internal/fix/filestore.go` (default dir `FixGWStore_<date>/`)
- **Mem store:** `internal/fix/memstore.go` (in-memory, dev-friendly)
- **SQL / Mongo:** configured via `MongoStoreConnection`, `SQLStoreDriver` — production uses MSSQL.

## Other Modules

- **Migrations:** `migrations/` (SQL)
- **REST API:** `internal/rest/` (Gin)
- **Domain / usecase:** `internal/domain/bond/usecase/`
- **DB repo:** `internal/infra/repo/bond/` (GORM)
- **Build:** `Makefile` → `tmp_bondgw.exe`

## Log Sample Reference

- **Location:** `c:\_project_git\Fix GW log\FixGWLog_<YYYY-MM-DD>\`
- **Files per session:**
  - `FIX.4.4-<sender>-HNX.messages.current.log` — raw FIX wire log
  - `FIX.4.4-<sender>-HNX.event.current.log` — session events
  - `GLOBAL.messages.current.log` / `GLOBAL.event.current.log`