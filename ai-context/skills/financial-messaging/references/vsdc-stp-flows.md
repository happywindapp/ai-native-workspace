# VSDC STP — Business Flows, BIC Codes, Folders, ACK/NAK

How carbon business operations map to MT message sequences, plus STP transport details.

## STP transport

VSDC connection is **file-polling**, not socket/HTTP. HSC installs a Gateway Client bridging BackOffice ↔ VSDC Gateway.

Root `D:\VSDClient\`:
| Folder | Role |
|---|---|
| `send` | BackOffice writes `.fin` here → Gateway pushes to VSDC |
| `receive` | Gateway writes VSDC output here → BackOffice reads |
| `archive` | Processed `.fin` / FileAct auto-moved here |
| `error` | Failed sends placed here by Gateway |

File types: `.fin` (1 ISO 15022 message), `.csv` (FileAct report data), `.par` (FileAct parameter file). `HSC_STP` polls `receive` every 15s.

## BIC codes

- TVLK BIC: `VSD` + 3-char member abbrev + `XX` (e.g. `VSDHSCXX`, `VSDSSIXX`, `VSDKLSXX`). If abbrev > 3 chars, drop trailing `X` (`VSDBVSCX`, `VSDACBSX`).
- **VSDC carbon market: `VSDSVN03`** — always the receiver for TVLK→VSDC carbon messages.
- Legacy bond market: `VSDSVN01` — NOT used for carbon.

## ACK/NAK

Service ID `21`. Block 4: `:177:` timestamp, `:451:` `0`=ACK/`1`=NAK, `:405:` error code.
- Protocol NAK: `H80` (delivery option), `[REQUESTID: duplicate]`.
- Business NAK: free-text in `:405:` (e.g. `TKGD … da co giao dich dang ky dang cho xac nhan`).

## Six business flows (HSC "Tên loại điện" taxonomy)

### 1. Đăng ký GD Carbon (register trading account)
1. TVLK→VSDC: **MT598.301** (`:22H::ACCT//AOPN`, `:22F::TPTY`, `:22F::ACTP`)
2. VSDC→TVLK: **MT598.301** reply (`:25D::IPRC//PACK`/`REJT`)
- MW endpoint: `POST /v1/stps/carbon/accounts/register`; status poll `/outputs/list?type=register-account`

### 2. Điều chỉnh loại hình TK (adjust account type quota↔credit)
1. TVLK→VSDC: **MT598.303** (`:22H::ACCT//MODE`, new `:22F::TPTY`/`ACTP`, `:95S::ALTE//VISD/…`)
2. VSDC→TVLK: **MT598.116** (`:25D::IPRC//ACPT`/`REJT`)
- MW endpoint: `POST /v1/stps/carbon/accounts/update`

### 3. Niêm yết / Hủy niêm yết (list / delist carbon code)
1. DCC→VSDC: **MT598.007** (quota 6-char) or **MT598.008** (credit 9-char)
2. VSDC broadcasts to all TVLK
3. Delist: DCC→VSDC **MT598.100** → VSDC→TVLK **MT546** (auto-withdraw)
- Phase 1: not implemented in MW — Terminal fetches product list from Info Service instead.

### 4. Lưu ký / Rút lưu ký (custody deposit / withdrawal)
1. DCC→VSDC: **MT540** (deposit) or **MT542** (withdrawal)
2. VSDC→DCC/TVLK: **MT544**/**MT546** (success) or **MT548** (reject)
- Phase 1: deferred (no carbon variant of MT540/542 templates yet).

### 5. Luồng thanh toán (settlement — core flow)
1. HNX→VSDC: trade result
2. If invalid → VSDC→TVLK(buy+sell): **MT598.000** (`:23G:CANC`)
3. VSDC→TVLK(buy+sell): **MT518** (payment obligation, `:22H::BUSE//BUYI`/`SELL`)
4. TVLK→VSDC: **MT598.305** (`:25D::STAT//CONF`/`REJT`) — confirm/reject obligation
5. VSDC→NHTT: **MT518** (bank payment obligation)
6. NHTT→VSDC: **MT900**/**MT910** (payment confirmed)
7. VSDC→TVLK(buy): **MT544** (credit); VSDC→TVLK(sell): **MT546** (debit)
8. Adjustment/removal: VSDC→TVLK **MT518** `:23G:CANC` with `:20C::PREV//`
- MW endpoints: `/payment-obligations/confirm`, `/payment-obligation/check`
- Carbon delivery only completes when both TVLK confirm MT598.305 + NHTT confirms MT900.

### 6. Thông báo phân bổ (allocation notification)
1. TVLK→VSDC: **MT598.010** (allocation data, `:70E::SPRO//` multiline)
2. Optional revoke: TVLK→VSDC **MT598.011**
- MW endpoints: `/v1/stps/carbon/allocation/{confirm,reject}`

### Supporting — EOD reports
VSDC→TVLK FileAct (`.par` + `.csv`) carrying MT598.005 (balance confirm) / MT598.006 (revoke). Report codes via `:13B::STAT//` (BS001, DE083, DE084…).

## Gateway constraints

- NĐT with a pending registration awaiting confirmation → cannot send another transaction on the same account → NAK.
- Remove-payment: if NHTT already confirmed cash → VSDC sends MT598 rejecting the withdrawal; if not → confirms it.

See `vsdc-mt-catalog.md` for message semantics, `mt-samples.md` for annotated samples.