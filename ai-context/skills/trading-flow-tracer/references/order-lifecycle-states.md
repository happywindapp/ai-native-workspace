# Order Lifecycle State Machine

Reference model for order/trade states. Adapt to the actual enum names found in the codebase (grep for `OrdStatus`, `OrderStatus`, `status` enums).

## Canonical states

| State | Meaning | Terminal? |
|---|---|---|
| `NEW` / `PENDING_NEW` | Created locally, not yet acknowledged by exchange | No |
| `PENDING` | Submitted, awaiting exchange/gateway ack | No |
| `ON_HOLD` | Held by risk/cash/limit check before release | No |
| `RELEASED` | Hold cleared, eligible to go to market | No |
| `OPEN` / `ACTIVE` / `NEW_ACK` | Acknowledged by exchange, live in book | No |
| `PARTIALLY_FILLED` | Some quantity matched | No |
| `FILLED` | Fully matched | Yes |
| `PENDING_AMEND` / `PENDING_REPLACE` | Amend submitted, awaiting ack | No |
| `AMENDED` / `REPLACED` | Amend acknowledged | No |
| `PENDING_CANCEL` | Cancel submitted, awaiting ack | No |
| `CANCELLED` | Cancel acknowledged | Yes |
| `REJECTED` | Rejected by validation/risk/exchange | Yes |
| `EXPIRED` | TIF expired | Yes |

## Legal transitions

```
NEW ──> PENDING ──> OPEN ──> PARTIALLY_FILLED ──> FILLED
  │        │          │            │
  │        │          │            └─> CANCELLED
  │        │          ├─> PENDING_AMEND ─> AMENDED ─> OPEN
  │        │          └─> PENDING_CANCEL ─> CANCELLED
  │        └─> REJECTED
  └─> ON_HOLD ──> RELEASED ──> PENDING ──> OPEN
        └─> CANCELLED   └─> REJECTED
```

## Amend with hold/release nuance

A common real-world flow: an amend on a live order may require the order to be **held**, the original **released/cancelled at the exchange**, then **re-submitted** with new terms. The state path is:

```
OPEN ─> PENDING_AMEND ─> (ON_HOLD) ─> release-original ─> re-submit ─> OPEN
```

Bugs cluster here: hold and release events arriving out of order, the re-submit firing before the release ack, or the amended order inheriting a stale ClOrdID.

## Invariants to check during a trace

- An order in a terminal state (`FILLED`/`CANCELLED`/`REJECTED`/`EXPIRED`) must receive no further mutating transitions.
- Every `PENDING_*` state must be followed by either its ack state or `REJECTED` — a `PENDING_*` with no resolution = stuck order.
- `filledQty` is monotonic non-decreasing and never exceeds `orderQty`.
- Each amend produces a new ClOrdID; `OrigClOrdID` chains back to the prior one.
- A `RELEASED` must be preceded by an `ON_HOLD` for the same order.

## Illegal transition = bug signal

If a trace shows e.g. `FILLED -> PENDING_CANCEL`, `ON_HOLD` with no prior hold trigger, or two concurrent `PENDING_AMEND` on one order — that is the divergence point. Record it.