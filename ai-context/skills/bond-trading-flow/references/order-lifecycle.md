# Bond Order Lifecycle

BondOMS handles BCGD Outright + TTDT Quotes. State lives in BondOMS Postgres (`order` table). Line numbers in `handler-api.go` drift — treat ±50 as the same site; grep the function name for canonical location.

## orderKind vs order_type vs action

These three are routinely confused. They are different things.

| Field | What it is | Values |
|---|---|---|
| `orderKind` | the *product/channel* | `"bondplus"` (real BondPlus UI user) / `"tprl"` (login-route caller left field blank — system/internal path) |
| `order_type` (column) | the *role of the DB row* | `order_root` / `order_edit` |
| action | the *operation* | place / amend / cancel / accept / match |

- BondPlus UI hardcodes `orderKind = "bondplus"`. The `/internal/commands/add` route validates `.isIn(["bondplus"])`.
- The login route `/commands/add` and `/commands/edit` do **not** validate `orderKind`; if the caller omits it, middleware falls back to `"tprl"`.
- Old Postman docs say FE sends `"order_root"` — **wrong**. That is the `order_type` column value, not `orderKind`.
- Cash-skip logic at `checkAccountOrder` skips the EQD pre-check when `orderKind == "tprl"`. This is the system/internal path (assumed pre-validated), **not** a skip for BondPlus user orders. BondPlus orders (`!= "tprl"`) still run the EQD pre-check.

### Two distinct FE workflows
| Screen | Endpoint | orderKind | Business meaning |
|---|---|---|---|
| `Home` | `/commands/add` (no orderKind) → `"tprl"` | broker-to-broker negotiated-trade report (deal agreed offline) |
| `BondPlus` | `/internal/commands/add` → `"bondplus"` | retail-style professional-investor self-service buy/sell |

## Order states (`hnx_status`)

| State | Value | Set in |
|---|---|---|
| NewOrder | 1 | `insertOrder` |
| Queue | 2 | `updateOrderFromHnx` |
| Canceled | 3 | `handleOne` ExecType=4 + `RetryUpdateHnx` non-BOND cancel |
| Rejected | 4 | `updateOrderFromHnx` |
| Pending_Edit | 5 | `AmendOrderHandler`, `RetryUpdateHnx`, `updateOrderFromHnx` |
| Completed | 6 | `handleOne` + `RetryUpdateHnx` case "3" |
| Pending_Confirm_Edit | 7 | `AmendOrderHandler`, `updateOrderFromHnx`, `updateOrderFromPartnerHnx` |
| Counterpart_Declined_Edit | 8 | `AcceptOrderHnxHandler`, `RetryUpdateHnx` case "4" |
| Declined_Edit | 9 | `handleOne`, `RetryUpdateHnx` case "9" |
| Pending_Cancel_Confirm | 10 | **never set — dead** |
| Counterpart_Declined_Cancel | 11 | **never set — dead** |
| Pending_Cancel | 12 | `AmendOrderHandler` cancel-via-edit only |
| Declined_Cancel | 13 | **never set — dead** |
| Cancel_Accepted | 14 | **never set — dead** |
| Invalid | 15 | `RetryUpdateHnx`, `updateOrderFromPartnerHnx` |

States 10/11/13/14 are dead — the cancel state machine is entirely unimplemented (see `known-bug-patterns.md`).

## Sync vs async — where state actually changes

- **Synchronous (gRPC dispatch):** `NewOrderHandler`, `AmendOrderHandler`, `CancelOrderHandler`, `MatchOrderHnxHandler` (counter-amend), `AcceptOrderHnxHandler` (counter-accept). These build/send FIX and insert/update rows.
- **Async (NATS consumer):** `SubscribeNatStream` → `handleOneSerial` (throttled ~1 msg/s) → `handleOne` switches on FIX MsgType. **`handleOne` only logs** — it writes `logs_hnx`, not the final `order` state.
- **Async batch state machine:** `RetryUpdateHnx` (15s cron) polls `logs_hnx` and applies the real transitions to `order`.

Consequence: there is an eventual-consistency window of seconds-to-minutes between an action and the visible state. "FE shows stale status right after an action" is by design (FE has no live data path either).

## ID columns — never overload

| Column | Meaning |
|---|---|
| `order_id_root` | original HNX OrderID — groups root + edit rows; set at INSERT, never overwrite |
| `order_id` | current HNX OrderID — changes on each amend; NULL between "user clicks Sửa" and HNX 39=A ack |
| `order_id_hnx` | the ClOrdID *sent* (BOND code or UUID) — NOT an HNX OrderID; critical for branch routing |
| `related_id` | same-firm pair grouping |
| `edited_id` | edit rows point to root's `related_id` |
| `order_id_match` | set when root MATCHED (real trade), independent of amend lifecycle |

`order_id_match != nil` means **the root matched a counterparty** — it does NOT mean "HNX confirmed the amend". HNX amend confirmation uses `hnx_status` transitions (ExecType=5).

## G3 orientation (Orient 1 / Orient 2)

A cross-firm row can be stored in two orientations. The handler must route the G3 op to the HSC account and flip `bid_ask` if needed. Wrong orientation → G3 no-op (hold on partner account) or wrong direction (cash instead of bond).

| Orientation | `order_company` | HSC position | `reciprocal_company` |
|---|---|---|---|
| **Orient 1** | `"011"` (HSC) | OrderAccount side | partner company |
| **Orient 2** | partner (`"009"`, etc.) | ReciprocalAccount side | `"011"` (HSC) |

- `bid_ask` always reflects the **OrderAccount** perspective, not HSC's.
  - Orient 1: `bid_ask` = HSC's side → use directly.
  - Orient 2: `bid_ask` = partner's side → **flip** when calling G3 ops on the HSC (reciprocal) side.
- Detection idiom: `if DataUpdate.OrderCompany == datastruct.Account_Company { /* Orient 1 */ }`.
- Origin of orientation per row: HSC-initiated (UI) → Orient 1; partner-initiated (`handleOne` case `s`) may keep the partner's FIX perspective → Orient 2. Post-match revert keeps the root row's orientation. One order lifecycle should not flip orientation mid-way.

## HNX FIX spec §3 → BondOMS coverage

| Spec | Scenario | Coverage |
|---|---|---|
| 3.3.10 | BCGD Outright same-firm (one-shot) | ✓ NewOrderHandler |
| 3.3.11 | BCGD Outright cross-firm (2-leg accept) | ✓ NewOrderHandler + counter-accept |
| 3.3.12 | Amend BCGD not-yet-executed | ✓ (OrdStatus=11 handling missing) |
| 3.3.13 | Cancel BCGD not-yet-executed | ⚠ dispatch only, no state |
| 3.3.14 | Amend BCGD executed, uncontrolled | ✓ mostly |
| 3.3.15 | Amend BCGD executed, HNX-controlled | ⚠ Pending_Confirm_Edit set; supervisor accept/reject partial |
| 3.3.16/17 | Cancel BCGD executed | ❌ cancel state machine dead |