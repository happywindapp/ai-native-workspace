# G3 Integration — Golden Rules

The rules for calling G3 *safely*. These cross-cut Bond and Carbon. Business-rule detail (which amend path, Carbon register flow) stays in `bond-trading-flow` / `carbon-trading-flow` — this file is the integration-call discipline.

## Rule 1 — Never call G3 inside a DB transaction

G3SB is a slow, failure-prone external SOAP call (60s timeouts). Holding a Postgres transaction open across a G3 call:
- pins a DB connection for the whole external round-trip,
- couples DB rollback semantics to an external system that has *already mutated state* if the call succeeded.

Carbon's arch review flags exactly this — "OMS G3+VCB in PG tx" — as a critical cross-cutting risk. Structure code so the external G3 call sits **outside** the tx boundary: either commit the DB write first then call G3, or call G3 first then write. The hold/release flows in BondOMS already largely do G3 calls in dedicated handler steps, not inside `tx`.

## Rule 2 — Hold(new) before release(old)

On any path that swaps a hold from `(price_A, qty_A)` to `(price_B, qty_B)` — amend submit, amend approve/reject reconcile, partner-initiated amend, post-match revert — **always hold the new amount first, release the old amount second.**

| Order | hold result | release result | Net state | Verdict |
|---|---|---|---|---|
| hold → release | OK | OK | correct | ✓ |
| hold → release | OK | **fail** | **over-hold** | safe — recoverable by manual release |
| hold → release | **fail** | (not run) | unchanged | safe — early-return on hold-fail, never run release |
| release → hold | OK | **fail** | **under-hold** | **DANGEROUS** — collateral leak, default-risk exposure |

Over-hold is always recoverable; under-hold can cause real loss if a counterparty defaults. Bias every swap toward over-hold.

Implementation discipline:
- On `hold` failure: log and `return` — do NOT run the release (leaves G3 state as it was before the swap).
- On `release` failure after a successful hold: log with an explicit `MANUAL INTERVENTION (over-hold)` marker so operations knows the cleanup scope.
- If old == new (no qty/price change), a `shouldSwitch`/`switched` check must return false so no G3 call is made at all.

```go
if h := holdAssetOrder(account, bidAsk, code, newP, newQ, fee); !h.Success {
    logs.Errorf("hold fail account=%s: %s", account, h.Error)
} else if rel := releaseAssetOrder(account, bidAsk, code, oldP, oldQ, fee); !rel.Success {
    logs.Errorf("release fail account=%s: %s - MANUAL INTERVENTION (over-hold)", account, rel.Error)
}
```

## Rule 3 — Account + bidAsk travel as a pair

A G3 hold/release call needs *both* the account and the `bidAsk` direction, and they must be **consistent**:
- `bidAsk = Buy` → cash hold/release.
- `bidAsk = Sell` → instrument (stock/bond) hold/release.

For a cross-firm row the stored account and the stored `bidAsk` are from the *order-side* perspective, not necessarily HSC's. When routing a G3 op to the HSC account on a reciprocal row, you must flip `bidAsk` together with swapping the account — pass the matched pair `(orderAccountCallG3, orderTypeCallG3)`, never a swapped account with a raw `DataUpdate.BidAsk`.

Mixing them flips cash↔instrument: the call either no-ops on the wrong account or holds the wrong asset (a known bond bug — `makeG3Order` Orient-2 bidAsk-not-flipped, and `bond-vsd-amend-orient-bug`). The Orient 1/2 *business matrix* lives in `bond-trading-flow`; the *call-level* rule is: account and bidAsk are one inseparable pair.

## Rule 4 — VALUEDATE = G3 core business date, not wall-clock

G3 core keeps its own internal business date and (especially on UAT) does **not** auto-roll it — it only advances when SOD (start-of-day) is run. Sending `VALUEDATE = time.Now()` when the core date is different triggers `ERROR_VALUE_DATE_IS_NOT_CURRENT_BUSINESS_DATE`.

- The canonical date lives in G3's DB table `HCBusinessDateToSystemTime`. Query:
  `SELECT TOP(1) businessDate FROM HCBusinessDateToSystemTime WHERE EndTime IS NULL ORDER BY StartTime DESC` (pool `dbG3SB` for equity / `dbG3FB` for derivatives). Same query CoreApiGateway's `getBusinessDate()` uses.
- Carbon-OMS env `ENABLE_G3_DATE=true` → `g3Date()` auto-fetches this, caches 60s, serialises concurrent callers with a mutex; on fetch failure it logs a warning and falls back to `time.Now()` (graceful degrade). `false`/unset → legacy `time.Now()`.
- `g3Date()` feeds **every** G3 date field — `VALUEDATE` (×4 in handler-core-api.go) plus `TRADEDATE` / `INSTRUMENTSETTLEDATE` / `CASHSETTLEDATE`, and 2 G3SB DB queries (account-contract, fee lookup). `TRANSACTIONREFERENCE` / `TRADEREFERENCE` keep `time.Now()` — they are identifiers, not business dates.
- **Owner of a date mismatch is the G3 core / back-office team** (roll the core date) — not OMS code, not DevOps. Auto-fetch is the defensive measure because "what Biz says the date is" and "what the core actually holds" have diverged in practice.

## Rule 5 — Idempotency & retry

- Idempotency key is `TRANSACTIONREFERENCE` (SOAP) / `TransactionNo` (g3sb-api.md). Re-sending the same reference → G3SB `30001 Transaction already exists` → **treat as success**.
- Retry only transient failures: G3SB `50001 System error` (after ~5 min) and `mode=1` connectivity errors (after infra is fixed). Retry with the *same* reference so `30001` makes it idempotent.
- Do NOT retry data errors (`10001`, `30002`, `40001`) or business fails (`20001`, `20002`) — no retry will change the outcome.
- Caveat: the OMS builds references with a random sequence (`yyyyMMddHHmmss{randSeq}`). A pure-random suffix is *not* a stable idempotency key across process restarts — a retry that regenerates the suffix will create a *new* hold, not reuse the old one. For true idempotent retry, persist the reference before the call and reuse it. (Bond `known-risks` flags "idempotency via random" as a latent risk.)

## Rule 6 — Trim config values

k8s/kustomize env files do not strip surrounding quotes the way `godotenv` does. A manifest value `G3SBApi_Url: "https://.../Endpoint"` can arrive with a literal trailing `"`, producing a request URL ending `/%22` → connection failure. Defensively `strings.Trim` G3 config (URL, creds) at startup and log the effective value. (Carbon-OMS does this in `main.go`.)

## Rule 7 — G3SB is raw SOAP; wrappers are OMS-side

G3SB exposes the raw `messageTransfer` SOAP operation only. There is **no** "wrapper API" inside G3. The `make*` helper functions, fee math, retry, and the hold/release abstractions all live in BondOMS / Carbon-OMS Go code. When debugging "the wrapper", look in the OMS repo, not G3.

## Rule 8 — `ENABLE_G3=false` is a test workaround only

Carbon-OMS env `ENABLE_G3=false` makes the OMS skip `checkAccountOrder` / `holdAssetOrder` / `releaseAssetOrder` / `makeG3Order` — so order flow can be exercised when G3SB infra is not ready. Orders placed in this mode have **no real hold backing them**. It is never a valid production state.

## Known CRITICAL integration bugs (catalog)

These are *G3-call-level* bugs. Bond-business-flow bugs stay in `bond-trading-flow`.

| # | Bug | System | Status | Note |
|---|---|---|---|---|
| 1 | NewOrder asset leak on gRPC/FIX fail — `success=true` set *before* `SendNewOrderCross`; fail path updates status without releasing the hold | BondOMS | STILL PRESENT | move `success=true` after the send/fix check |
| 2 | Cancel releases G3 hold *before* HNX confirms — release → tx.Commit → cancel gRPC; HNX reject leaves the asset already released (double-spend) | BondOMS | STILL PRESENT | defer release to ExecutionReport `150=4, 39=3` |
| 3 | Dual-hold on amend | BondOMS | FIXED 2026-04-17 | "hold vừa đủ cover" rule (business detail → `bond-trading-flow`) |
| 4 | `makeG3Order` cross-firm `bidAsk` not flipped (Orient-2) → cash↔bond inverted, G3 reject | BondOMS | FIXED 2026-04-28 | now uses `orderTypeCallG3` — Rule 3 |
| 5 | `G3SBApi_Url` stray trailing `"` from k8s env → URL `/%22` → connection refused | Carbon-OMS | FIXED 2026-05-14 | `strings.Trim` at startup — Rule 6 |
| 6 | `VALUEDATE = time.Now()` ≠ core business date → `ERROR_VALUE_DATE...` | Carbon-OMS | mitigated 2026-05-15 | `ENABLE_G3_DATE` auto-fetch — Rule 4 |
| 7 | Cancel branches update DB status but never release the G3 hold | BondOMS | TODO | `RetryUpdateHnx` cancel branches — needs data test |
| 8 | Idempotency key built from random suffix — not stable across retries | BondOMS | latent risk | Rule 5 caveat |

> Line numbers for any of these drift. Grep the function name (`makeCashHold`, `makeG3Order`, `holdAssetOrder`, …) in the relevant repo before relying on a location.