# BondTradingMiddleware Map — Routes, Services, Jobs

Node 16 / Express / Sequelize-MySQL. Stateful orchestrator between BondTerminal_FE and BondOMS/HSC_STP — not a thin BFF. Marked for retirement (wrong architectural boundary).

> As-of: built 2026-04-17, re-verified 2026-05-14. A prior 2026-04-23 note claiming `/match`, `/accept`, `getMyPerspective` were removed was WRONG — they still exist.

## API Routes — commands (`/api/commands`)

| Method | Path | Controller | Purpose |
|---|---|---|---|
| POST | `/add` | addNew | Create command → OMS `/new` |
| POST | `/edit` | editCommand | Edit command → OMS `/amend` |
| GET | `/list` | getListCommands | List commands → OMS `/list` |
| GET | `/comparison` | getComparison | OMS summary (dataOMS, dataG3) |
| GET | `/list-comparison` | getListComparison | Detailed OMS/G3/VSD comparison |
| DELETE | `/cancel` | cancelCommand | Cancel → OMS `/amend` (hnxStatus=cancel) |
| DELETE | `/delete` | deleteCommand | Delete → OMS `/cancel` |
| GET | `/list-confirm` | getListConfirm | Today's bondDeposit confirm/cancel records |
| POST | `/confirm` | confirmCommand | Allocation confirm → STP `/allocation/confirm` |
| POST | `/cancel-confirm` | cancelConfirmCommand | Allocation reject → STP `/allocation/reject` |
| GET | `/:commandId?` | getCommand | Single command detail |

`/match` and `/accept` routes still exist (`route/command.js` ~:24-32) → controllers `matchCommand`, `acceptCommand`.

## API Routes — quotes (`/api/quotes`)

TTDT quote routes (`route/quote.js`) — all POST (cancel uses POST not DELETE to avoid gateway stripping the body):
`POST /add`, `POST /edit`, `POST /cancel` (~:12), `POST /accept`, `GET /list` → forward to OMS `/equity/quote/*`.

## API Routes — accounts (`/api/accounts`)

| Method | Path | Controller | Purpose |
|---|---|---|---|
| POST | `/add` | addNew | Register bond account (MT598 → STP) |
| POST | `/edit/:accountId?` | amendAccount | Update account (MT598 → STP) |
| POST | `/amend/:accountId?` | editAccount | Mark status=completed |
| POST | `/checkList` | getList | Paginated list (POST body) |
| GET | `/update-account` | getList | Update-account list |
| DELETE | `/:accountId?` | deleteByAccountId | Delete account |
| GET | `/list` | getList | Account list (paginated) |
| GET | `/:accountId?/info/:type?` | getAccount | Account detail from G3B |

## API Routes — internal (`/api/internal`, for BondPlus)

`POST /commands/add` (addNew), `GET /commands/list` (getListCommands), `POST /commands/pull-status` (pullStatusCommands).

## API Routes — other

| Path | Controller | Purpose |
|---|---|---|
| GET `/api/bonds` | getListBond | Bond list from Info service |
| POST `/api/bonds/update-account` | getUpdateAccount | KRX STP account updates |
| POST `/api/login` | login | Azure token verify → JWT |
| `/api/users` | CRUD | User management |
| GET `/api/members` | — | Trading members list |
| GET `/api/logs` | — | Audit logs |
| GET `/api/info/report-vsd` | getReportVSD | VSD trade statements |
| POST `/api/info/edit` | editCommand | Edit via info service |
| GET `/api/info/balance/:id` | getBalance | Account balance from OMS |

## Services — `handleResFromSTP.js` (STP gateway, ~1252 lines)

| Line | Function |
|---|---|
| 33 | callToAddAccountSTP — register account → STP |
| 107 | callToUpdateAccountSTP — update account → STP |
| 171 | callToConfirmPaymentSTP — confirm/reject payment (is_confirm=T/F) |
| 220 | callToCheckSHL |
| 276 | runGet598STP — Job: register-account outputs |
| 337 | runGetUpdateSTP — Job: update-account outputs |
| 397 | runGetStatusVSDFromSTP — Job: main reconciliation (518+544+546) |
| 953 | runGetDeniedSTP — Job: trade-error outputs |
| 1023 | runGetCancelSTP — Job: trade-cancel outputs |
| 1093 | runSendConfirmSTP — allocation confirm → STP `/allocation/confirm` |
| 1161 | runSendCancelConfirmSTP — allocation reject → STP `/allocation/reject` |
| 1218 | callToGetTradeStatements |

`getMyPerspective` (from `functions/get-my-perspective`, imported ~:19) is used at ~:567/711/824/852/882/919 for MT518/544/546 cross-firm bidAsk swap.

## Services — `handleResFromOMS.js` (OMS gateway, ~584 lines)

| Line | Function | OMS endpoint |
|---|---|---|
| 11 | callToGetComparison | GET `/summary` |
| 45 | callToGetListComparison | GET `/comparison` |
| 79 | callToGetOne | GET `/list?id=` |
| 113 | callToGetList | GET `/list?params` |
| 152 | callPullStatusCommands | POST `/list-status` |
| 188 | callToAddNew | POST `/new` |
| 226 | callToEditCommand | POST `/amend` |
| 263 | updateHistory | POST `/update-history` |
| 300 | editCommand | POST `/edit` |
| 337 | getBalance | GET `/vcb/balance/{id}` |
| 371 | callToAcceptCommand | POST `/accept` |
| 408 | callToMatchCommand | POST `/match` |
| 445 | callToCancelCommand | POST `/cancel` |

## Services — other

- `handleResFromInfo.js`: `callToGetBond()` → Info service `/api/privateBond`.
- `handleResFromKrxStp.js`: `callToGetUpdateAccount()` → KRX STP `/update-account/list`.
- `account.js`: `getExistAccount`, `checkExist`, `getByAccountId`, `getAllCompleted`, `getInfoAccount` (G3B calls).

## Scheduled jobs

Trigger: `GET /commands/call-update` (header `X-Api-Key: bond-terminal`). External scheduler hits it periodically; runs 5 jobs sequentially:
1. `runGet598STP()` — register-account outputs.
2. `runGetUpdateSTP()` — update-account outputs.
3. `runGetDeniedSTP()` — trade-error outputs.
4. `runGetCancelSTP()` — trade-cancel outputs.
5. `runGetStatusVSDFromSTP()` — main reconciliation (598 confirm + 518 + 544 + 546).

## Data models

| Model | Key fields | Purpose |
|---|---|---|
| Commands | bondCode, shl, matchSHL, rootSHL, statusHNX, statusVSD, typeTT, messageTransfer(JSON), history(JSON) | Order/command record |
| Accounts | accountId, status(inProgress/completed/cancel), transactionCSVId | VSD custody account |
| BondDeposit | transactionId, type(confirm/cancel), status(success/fail), trade_date, payment_date, total_value | Allocation history |
| CheckConnect | orderId, shl, status | Idempotency for 598 confirm |
| Logs | from(STP/OMS/JOB), shl, des(tracking key) | Audit log |
| Users | username(email), role, deleted | User management |
| Members | code, memberName | Trading members |

## Auth & config

- Azure AD token verify → JWT (`login.js`).
- `accessMiddleware.js`: Bearer JWT → `res.dataUser {mail, id, name, location, displayName}`.
- G3B middleware enriches `req` with accountInfo/Cash/Asset from OMS.
- `constants/index.js`: service URLs (`urlAccountService`=OMS, `urlSTPService`, `urlInfoService`, `urlKrxStpService`), `limitCashAccount=2,000,000,000`, bond key `"bond-terminal"`, `defaultOrderKind: "tprl"`.