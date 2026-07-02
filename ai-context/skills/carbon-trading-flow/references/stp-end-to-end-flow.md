# Carbon STP End-to-End Flow

The STP pipeline carrying VSDC SWIFT MT messages: Carbon-OMS ⇄ Carbon-Middleware ⇄ HSC_STP ⇄ VSDC.
Verified 2026-05-12 from source + `documents/Bộ điện mẫu Carbon STP.xlsx`. For MT message *format* see `financial-messaging`.

## Topology

```
Terminal ──HTTPS/JWT──▶ Middleware (Node 8820) ──HTTP──▶ OMS (Go 3000)   [order/cash/asset]
                                              └──HTTP──▶ HSC_STP (Go 3006) [VSDC MT]
                                                              │
                                                    file polling 15s
                                                              ▼
                                                  D:\VSDClient\{send,receive,archive,error}
                                                              ▲
                                                       VSDC Gateway Client
                                                              ▲
                                                     VSDC (BIC VSDSVN03)
```

No Kafka, no WebSocket, no reverse webhook — everything is **REST + file polling**.

## Channels

| Hop | Protocol | Auth | Notes |
|---|---|---|---|
| Terminal → Middleware | HTTPS REST | Entra ID JWT | sync request |
| Middleware → OMS | HTTP REST (`192.168.82.66:3000`) | none (K8s netpol) | sync, NO callback — MW must poll |
| Middleware → STP | HTTP REST (`192.168.82.69:3006`) | header `x-api-key: carbon-terminal` + lane `client-id` | sync trigger send; status via poll `/outputs/list?type=…` |
| OMS → STP | **none** — OMS never calls STP directly | — | — |
| OMS → G3 Core | SOAP XML | CompanyID/User/Password header | hold/release cash+asset, import trade |
| OMS → VietinBank | REST + RSA-2048 | signed payload + session key | remittance-add, query status (cron 15s) |
| OMS → SmartNotify | REST | — | email on tx success |
| STP → VSDC | file polling `.fin` ISO15022 | — | bidirectional via folders |
| STP → iTrade | REST | — | get account info before generating MT |
| STP → SmartNotify | REST | — | email on inbound MT |

## Lane resolver — Bond vs Carbon in HSC_STP

`HSC_STP` is shared. Each MW request carries a lane via the `MiddlewareFromService` middleware:
- `/v1/stps/private-bond/*` → lane `bond` → legacy TPRL template (BIC `VSDSVN01`)
- `/v1/stps/carbon/*` → lane `carbon` → Carbon template (BIC `VSDSVN03`)

Template selection by resolver helper:
- `resolveAccountTemplate()` (send_stp_account.go): `from_service=carbon` → `TemplateCarbonRegisterAccount` / `TemplateCarbonUpdateAccount`
- `resolveAllocationTemplate()` (stp_allocation.go) → `TemplateCarbonConfirmAllocation` / `TemplateCarbonRejectAllocation`

> The service deploys as `bond-stp` (namespace `private-bond-uat`) and reads the shared `/g3sb-csd-receive/` folder for both projects — routing is at the URL/handler layer, not separate folders. Debugging Carbon STP means reading `bond-stp` logs filtered by `/v1/stps/carbon` or carbon order IDs (`{YYYYMMDD}-TESTxx`).

## Endpoint → template → MT mapping (Middleware → STP)

| Action | Endpoint | Carbon Template | MT subtype |
|---|---|---|---|
| Register Carbon account | `POST /stps/carbon/accounts/register` | `TemplateCarbonRegisterAccount` | MT598.301 NORMAL (AOPN/ACLS) |
| Update account info | `POST /stps/carbon/accounts/update` | `TemplateCarbonUpdateAccount` | MT598.303 MODE |
| Confirm KQGD | `POST /stps/carbon/payment-obligations/confirm` | `TemplateConfirmPaymentObligation` ⚠️ | spec wants MT598.305 TRADE — code still sends 598.222 (S1) |
| Confirm cash/credit allocation | `POST /stps/carbon/allocation/confirm` | `TemplateCarbonConfirmAllocation` | MT598.308 CASH |
| Reject allocation | `POST /stps/carbon/allocation/reject` | `TemplateCarbonRejectAllocation` | MT598.309 CASH (REJT) |

Deposit/withdraw (MT540/MT542) have **no Carbon variant yet** — only Bond templates exist (D2 deferred).

## Canonical Terminal → MT tag mapping (Carbon account)

| Terminal field | Value | MT tag | MT value | Meaning |
|---|---|---|---|---|
| `investor_type` | 1 | `:22F::TPTY//` | `EMIT` | emitting facility |
| `investor_type` | 2 | `:22F::TPTY//` | `PROJ` | credit-project owner |
| `investor_type` | 3 | `:22F::TPTY//` | `ORGA` | other org |
| `accountTypeName` | `quota` | `:22F::ACTP//` | `QUOT` | Hạn ngạch |
| `accountTypeName` | `credit` | `:22F::ACTP//` | `CRDT` | Tín chỉ |
| `memberType` | 3-7 | `TYPE//` (in 70E::ADTX) | DOMIND/FORIND/DOMCORP/FORCORP/GOVT | account class |

Mapping is applied at MW (`handleResFromSTP.js` top-level `tptyTypeMap`/`actpTypeMap`) before sending to STP. To extend a business rule, update those 2 maps.

## Outbound pipeline (TVLK → VSDC)

1. MW POST `/stps/carbon/<action>` with `client-id` header.
2. Handler binds request, `MiddlewareFromService("carbon")` sets the lane context.
3. Usecase `Input.ToSendStpInput()` builds the tag list + `transactionRef = HSC{unixMilli}`.
4. `resolveXxxTemplate()` selects the Carbon or Bond template.
5. `ReplaceTemplate()` substitutes `{Tag_…}`.
6. `sendStp()` writes file `{SendFolder}/{unixMilli}.fin` (retry 3× × 200ms).
7. `logSendStp()` inserts `stps_input` with `from_service=carbon`; `storeSendStp()` inserts `send_stp`.
8. VSDC Gateway Client pushes to VSDC and drops ACK/NAK files into the receive folder.

## Inbound pipeline (VSDC → TVLK)

1. Cron `@every 15s` runs `ReadStpReceive()` + `ReadParReceive()`.
2. `filepath.Glob(ReceiveFolder/*.fin*)` collects files.
3. DB lookup `stps_input.reply_file_name` + `stps_output.file_name` → skip already-processed.
4. Each new file: `ReadStp()` → `stp.Parse()`:
   - block 1 header `F21` → ACK/NAK reply → `parseReply()`.
   - block 1 header `F01` → new output → `parseOutput(stpType)`.
5. `LinkStpReceiveWithSend()` maps ACK/NAK back to a `send_stp` record.
6. `StoreStp()` inserts `stps_output` + the specific MT table (`stps_o518/o544/o546/o548/...`) + updates `send_stp.accept = DONE|FAIL`.
7. `SendNotificationStp()` emails via SmartNotify.
8. Retry counter `stps_input.retry_get_reply++` up to `MaxRetryGetReply=1000`, then drop.

> Known quirk: the move-to-processed-folder code (`stp.go:274`) is commented out — files stay in the receive folder, polling re-checks DB each tick.

## MT inbound parser coverage

`stp.Parse()` switches on the first 4 chars of the MT type.

| MT | Parsed? | Purpose |
|---|---|---|
| O598 | ✅ `parseO598` | confirm open/close/modify account, cancel symbol reg, broadcast new symbol |
| O518 | ✅ `parseO518` | KQGD + payment obligation |
| O544 | ✅ `parseO544` | confirm credit-increase / buy-side settlement |
| O546 | ✅ `parseO546` | confirm credit-decrease / sell-side settlement |
| O548 | ✅ `parseO548` | reject custody/withdraw |
| O564/O567/O568 | ✅ | rights notification / allocation confirm / additional |
| I598 | ✅ `parseI598` | input from VSDC bank (legacy) |
| O508 | ❌ no parser | confirm credit freeze/unfreeze |
| O900/O910 | ❌ no parser | confirm cash payment |

Unparsed MT types → `parseOutput()` returns `"not defined"`, file is skipped, retries to `MaxRetryGetReply=1000`.

## How Middleware gets results

MW receives no callback — it **polls** STP `GET /stps/carbon/outputs/list?type=<stpType>&ref=<msgRef>`. By `request.Type`: `register-account`/`update-account` → `stps_o598`; `payment-obligation` → `stps_o518`; `increase-amount` → `stps_o544`; `decrease-amount` → `stps_o546`.

## "Loại điện" taxonomy — 6 business groups

HSC business/ops users classify all STP/VSDC messages into 6 groups (different from the tech-side MT-code naming). Always tag a "Tên loại điện" column when listing STP APIs.

| # | Tên loại điện | MT messages | MW endpoints |
|---|---|---|---|
| 1 | Đăng ký GD Carbon | MT598.301 | `/stps/carbon/accounts/register`, `/outputs/list?type=register-account` |
| 2 | Điều chỉnh loại hình TK | MT598.303 | `/stps/carbon/accounts/update`, `/outputs/list?type=update-account` |
| 3 | Niêm yết mới / Hủy niêm yết | MT505/506 (Carbon variant unconfirmed) | **not implemented** in MW — Info Service fetches product list instead |
| 4 | Lưu ký / Rút lưu ký | MT540/542 | **not implemented** (D2 deferred) |
| 5 | Luồng thanh toán | MT518, MT544/546, MT598.confirm, trade-error/cancel/statements | `/payment-obligations/confirm`, `/payment-obligation/check`, `/outputs/list?type={payment-obligation,increase-amount,decrease-amount,trade-error,trade-cancel}`, `/trade-statements/list` |
| 6 | Thông báo phân bổ | allocation confirm/reject (custom Carbon REST) | `/stps/carbon/allocation/{confirm,reject}` |

## Known code-vs-spec gaps (as of 2026-05-13)

These are *stable structural* facts; dated incident detail lives in project memory.

| # | Service | Issue | Status |
|---|---|---|---|
| **S1** | HSC_STP | `TemplateConfirmPaymentObligation` has no Carbon variant — hardcodes `:12:222` (legacy TPRL) + BIC `VSDSVN01`. Spec needs `:12:305` `:77E::PROC//TRADE` + `VSDSVN03`. → `/stps/carbon/payment-obligations/confirm` is rejected by VSDC. Needs `TemplateCarbonConfirmPaymentObligation` + `resolveConfirmPaymentObligationTemplate()`. | ❌ **Phase-1 BLOCKER** |
| S2 | HSC_STP | Carbon route group missing `GET /stps/carbon/payment-obligation/check` → MW `callToCheckSHL` 404 → false-negative dedup. | ❌ |
| S3 | HSC_STP | Parsers O508/O900/O910 not implemented. | ❌ (settlement) |
| D1 | HSC_STP | Close-account flow (`:22H::ACCT//ACLS`) — register template hardcodes `AOPN`. | 🟡 deferred (Phase 1 no close) |
| D2 | HSC_STP+MW | MT540/542 Carbon deposit/withdraw not wired (~9 changes across 3 services). | 🟡 deferred (Phase 2) |

The MW-side fixes (M1+M2+M4+M5 — TPTY/ACTP mapping, KYC forwarding, carbon-terminal note) and STP-side S6/S7/S8 (subtype filters `IN ('201','301')` / `IN ('203','116')`, 70E::REGI source) were already done 2026-05-12/13.

## References

- Excel spec: `c:\_project_cabon\documents\Bộ điện mẫu Carbon STP.xlsx` (6 sheets).
- Go templates: `HSC_STP/internal/domain/hsc_stp/constant/template.go`.
- Pipeline: `HSC_STP/internal/domain/hsc_stp/usecase/stp.go`.
- Router: `HSC_STP/handler/configure.go` (carbon group ~line 50).