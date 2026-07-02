# QTTT — Quy Trình Thanh Toán TPRL

Canonical VSD–TVLK–NHTT–SGDCK MT message flows for the TPRL settlement process. Source: `Danh Sách Điện STP - TPRL.xlsx` sheet "Flow QTTT". This is the business-flow view; MT wire format belongs to `financial-messaging`.

**Actors:** VSD, TVLK (HSC), NHTT (settlement bank), SGDCK (HNX), counterparty TVLK.

## Flow 1 — Register / de-register a TPRL custody account
TVLK ↔ VSD: **MT598** request (1) / **MT598** response (2).

## Flow 2 — Adjust TPRL investor information
TVLK ↔ VSD: **MT598** request (1) / **MT598** response (2).

## Flow 3A — TPRL settlement, HNX result pushed to VSD
1. SGDCK → VSD: receive TPRL trade result (KQGD).
2. VSD → TVLK: **MT598** KQGD invalid (reject path).
3. VSD internal: validate account, check bond balance, lock TPRL.
4. VSD → TVLK: **MT518** payment obligation.
5. TVLK → VSD: **MT598** confirm obligation.
6. VSD → NHTT: **MT518** payment obligation.
7. TVLK → NHTT transfers cash; NHTT → VSD **MT910** cash-payment confirm.
8. VSD: settle the bond.
9. VSD → TVLK (buy side): **MT544** increase TPRL.
10. VSD → TVLK (sell side): **MT546** decrease TPRL.
11. TVLK → VSD: **MT598** withdraw the settlement obligation (exception path).
12. VSD → TVLK: **MT598** confirm withdrawal.
13. VSD → TVLK: **MT598** reject withdrawal.
14. VSD ↔ TVLK: **MT598** reconcile message count.
15. VSD → TVLK: **FileAct + MT598** consolidated KQGD report.

## Flow 3B — TPRL settlement, off-exchange VSD transfer with cash settlement
VSD enters off-exchange from documents → (4) **MT518** to TVLK → (5) **MT598** confirm → (6) **MT518** to NHTT → cash transfer → (7) **MT910** confirm → bond transfer → (9) **MT544** to receiver → (10) **MT546** to sender. No steps 2/3 (no invalid-KQGD path, since it is entered manually).

## Flow 4 — Remove a TPRL trade settlement
VSD → **both** TVLKs (the member and the counterparty): **MT518** removal notice.

## Flow 5 — Trade-error handling
VSD → both TVLKs: **MT518** KQGD-adjustment notice.

## Flow 6 — Notify cash/TPRL allocation to the investor
TVLK → VSD: **MT598** allocation notice (1) / **MT598** cancel allocation confirmation (2).

## Flow 7 — TVLK / TCMTKTT reconciliation
(15) VSD → TVLK: **MT598** (report code) consolidated KQGD report + **FileAct (.csv)**.

## Gotchas

- MT598 step 11 (withdraw obligation) is a TVLK-initiated exception path — middleware must distinguish it from a normal MT598 confirm.
- Flow 3B skips steps 2/3 (no invalid-KQGD because it is entered by hand).
- Flow 4 and Flow 5 broadcast MT518 to **both** TVLKs — the dispatch must guarantee both receive it.