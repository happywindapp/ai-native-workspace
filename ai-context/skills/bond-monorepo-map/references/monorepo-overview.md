# Monorepo Overview — `c:\_project_git`

The HSC Vietnamese private bond (Bond Riêng Lẻ / TPRL / BondPlus) trading monorepo — 7 sub-projects, integrating HNX via FIX 4.4 and VSD via SWIFT MT files.

> As-of: layout verified 2026-04-13 → 2026-05-14. Grep to confirm exact locations.

## Projects

| Path | Stack | Role / Key facts |
|---|---|---|
| `BondFIXOrderGW/` | Go 1.23, quickfix-go (HscQuickfix fork), NATS JetStream | FIX 4.4 gateway to HNX. Mirrors all FIX msgs → NATS. Thin relay — business logic lives in NATS consumers (BondOMS). |
| `BondOMS/` | Go, Fiber, gRPC; dual DB — Postgres `dbpg` (owns `order`/`logs`/`log_hnx`) + MSSQL `dbG3SB` (read-only mirror of G3SB) | Core OMS. Owns `hnx_status`+`vsd_status`+`bank_status`. 6000+ line monolith `handler-api.go`. Calls BondFIXOrderGW via gRPC. SOAP to G3SB for hold/release. No tests, no alerting infra — only `log.Printf`/`logs.Warnf`. |
| `BondTradingMiddleware/` | Node 16, Express, Sequelize/MySQL | Stateful orchestrator (not a thin BFF despite the name). Owns VSD status polling via STP. Marked for retirement in the 2026-04-13 audit — wrong architectural boundary. |
| `BondTerminal_FE/` | Next.js 13 Pages Router, React 18, Redux-Saga, MUI v5, MSAL (Azure AD) | Trader UI. Calls BondTradingMiddleware + BondOMS. No live push — fetch-on-mount + interval polling. |
| `HSC_STP/` | Go 1.19, gin, gorm | SWIFT MT transport (o518/o544/o546/o548/o564/o567/o568/o598/i598). Writes `.fin` files, cron-polls receive folder every 15s. Transport-only — does NOT own `vsd_status`. |
| `HSC_STP_ADMIN/` | Next.js 9.1.5, React 16, LDAP+JWT | Admin console for STP operators. Severely outdated 2019-era stack. |
| `HscGoModules/` | Go 1.24 shared libs | cache / concurrency / contxt / crypto / ginwrapper / http / infra / middleware / project. Consumed by BondOMS, HSC_STP, BondFIXOrderGW. The `crypto` package is dangerous (AES-CBC uses key as IV, ECB exposed). |

## FIX-gateway service layout (structural)

- `BondFIXOrderGW/` is the FIX 4.4 gateway to HNX for TPRL. Phase 1 went live 2023-07-21 (manual entry); Phase 2 (auto-sync BT → HNX + auto status) in progress.
- It is a thin relay: it mirrors every FIX message it sees onto NATS JetStream; the actual business handling runs in NATS consumers inside BondOMS.
- `BondFIXOrderGW/internal/client/in_msg_hdl.go` contains stub inbound handlers — these are **dead code**, not the live path.
- Detailed FIX code map kept separately in memory `bondfixordergw-code-map.md` (HnxQuickfix lib, fix44 structs, `sendMsgToNats` entry, config) — FIX *protocol* details belong to `financial-messaging`.

## Data flow — happy path for a new bond order

```
BondTerminal_FE (MSAL Azure AD auth)
   ↓ HTTPS
BondTradingMiddleware (Node orchestrator)
   ↓ axios REST
BondOMS (Go/Fiber)  ──── owns hnx_status 1-15, vsd_status 1-10, bank_status 1-5
   ├── gRPC → BondFIXOrderGW ── FIX 4.4 → HNX
   │              └── mirror all FIX msgs → NATS JetStream → consumers (in BondOMS)
   ├── SOAP → G3SB (asset holds / pre-trade checks)
   └── HTTP → HSC_STP ── SWIFT MT .fin files → VSD (settlement)
                 ↑
            HSC_STP_ADMIN (legacy admin UI)
```

## NATS ingress path (critical)

- NATS JetStream is the real FIX ingress into BondOMS — not the stub handlers in BondFIXOrderGW.
- Consumer: `BondOMS/handler-api.go` → `SubscribeNatStream`, spawned as a background goroutine from `BondOMS/main.go` (~`:191`).
- Subject pattern: `{NATS_STREAM_SUBJECT}-{YYYY-MM-DD}.fix44`.
- Routes by `data["MsgType"]` string into a switch (cases `"s"`/`"t"`/`"8"`/`"3"`/`"AI"`/`"AJ"` etc.); writes DB via `insertLogHnx`.
- NATS reconnect handled by the `RetrySubscribe` cron (see `bondoms-map.md`).

## Cross-module facts

- `hnx_status`/`vsd_status` constants live in `BondOMS/datastruct/constant.go` (~`:5-37`) and match `docs/Bond Riêng Lẻ - status.md` exactly.
- `hnx_status` and `vsd_status` are two parallel, independent state machines — never merge. They can be in different states simultaneously (HNX done + VSD reject).
- BondTradingMiddleware writes `vsd_status` back to BondOMS via 5-step STP polling → known race with BondOMS's own writes (no resolution yet).
- HSC_STP is transport-only — it never references the `vsd_status` enum.
- Only `hnx_status` values 2/3/4/6/7 are real FIX events written to `logs_hnx`.
- Every project has `azure-pipelines.yml` → CI is Azure DevOps. Most projects ship a committed `Dockerfile`.

## Key docs (in `c:\_project_git\docs\`)

| Doc | Content |
|---|---|
| `Bond Riêng Lẻ - status.md` | Authoritative state-machine spec: FIX msg types (tag 35), `hnx_status` 1-15, `vsd_status` 1-10, ~180 HNX error codes (`Desc_Code_Hnx`). |
| `README_BOND.md` | Architecture overview. |
| `BondTradingSystem_Architecture.docx` | Binary — cannot read directly; use office-extract. |

## Audit reports (`c:\_project_git\plans\reports\`)

- `review-260413-0228-{project}.md` — shallow reviews of all projects.
- `deep-review-260413-0228-{project}.md` — deep reviews of bondoms, bondfixordergw, bondtradingmiddleware, bondterminal-fe.
- `fix-protocol-spec-260413-0228-cbts-v115.md` — CBTS FIX v1.15 catalog.
- `fix-flow-deep-260414-0513-create-modify-cancel.md` — sequence diagrams.
- `nats-consumer-trace-260414-0513-bond-fix-pipeline.md` — NATS consumer location, 1 msg/s throttle, orphan DLQ.

Verify findings via grep before acting — issues may already be fixed.