# G3 / G3SB Error Codes

Two error layers:
1. **G3SB numeric codes** — returned in the SOAP response body (`ErrorCode` + `ErrorMessage`). From g3sb-api.md v1.09.
2. **Named G3-core errors** — string error tokens the G3 core itself returns (e.g. `ERROR_VALUE_DATE_IS_NOT_CURRENT_BUSINESS_DATE`).

Plus the **OMS call-mode classification** — how the OMS code categorises *where* a hold call failed.

## 1. G3SB numeric error codes (g3sb-api.md v1.09)

| Code | Meaning | Reaction |
|---|---|---|
| `00000` | Success | proceed |
| `10001` | Account not found | data fix — wrong/unregistered `ACCOUNTID`; do NOT retry |
| `10002` | Account inactive / frozen | data/ops fix — account state issue; do NOT retry |
| `20001` | Insufficient balance | business fail — surface to caller (e.g. `holdCashFail`); do NOT retry |
| `20002` | Hold exceeds balance | business fail — same as 20001 |
| `30001` | Transaction already exists | **idempotent reuse — treat as success** for the same `TRANSACTIONREFERENCE`. Not a hard error. |
| `30002` | Invalid amount / currency | data fix — check `AMOUNT` format / `Currency=VND`; do NOT retry |
| `40001` | Contract not found | data fix — account has no contract; create contract first |
| `40002` | Contract inactive | ops fix — contract state issue |
| `50001` | System error | **retry after ~5 min** — transient G3 issue |
| `99999` | Unhandled error | escalate — unknown; capture full response, raise to G3/back-office team |

Reaction classes:
- **Retry** — `50001` (transient). Retry with the *same* `TRANSACTIONREFERENCE` so `30001` makes the retry idempotent.
- **Idempotent success** — `30001`. The hold/release already landed; continue.
- **Data fix** — `10001`, `30002`, `40001`. Caller sent bad input; no retry will help.
- **Business fail** — `20001`, `20002`. Genuine insufficient funds/inventory; surface to user.
- **Ops/state fix** — `10002`, `40002`. Account/contract needs back-office action.
- **Escalate** — `99999`.

## 2. Named G3-core errors

| Token | Meaning | Reaction |
|---|---|---|
| `ERROR_VALUE_DATE_IS_NOT_CURRENT_BUSINESS_DATE` | `VALUEDATE` sent ≠ G3 core's internal business date. G3 core (esp. UAT) does not auto-roll its date; SOD (start-of-day) may not have run. | Send `VALUEDATE` = real core business date. Carbon-OMS `ENABLE_G3_DATE=true` auto-fetches it. **Owner = G3 core / back-office team** to roll the date — not OMS code, not DevOps. See `integration-rules.md` Rule 4. |

> The named-token error space is larger than this one entry — only this token is documented in current memory (Carbon UAT incident, 2026-05-14). When an unknown named token appears, capture it verbatim and route to the G3/back-office team.

## 3. OMS call-mode classification (Carbon-OMS `makeCashHold` instrumentation)

When `makeCashHold` fails, Carbon-OMS logs a `mode` telling *where* in the call the failure happened:

| Mode | Meaning | Owner / reaction |
|---|---|---|
| `mode=1` http-post-error | SOAP HTTP POST itself failed — e.g. `dial tcp 192.168.x.x:8080: connect: connection refused`. G3SB endpoint unreachable / wrong host:port. | **DevOps + G3SB team** — infra. Not a code fix. Check `G3SBApi_Url` is correct for the environment and that something is listening. |
| `mode=4` core-rejected | HTTP POST succeeded; G3 core returned an error in the response body (e.g. `ERROR_VALUE_DATE_IS_NOT_CURRENT_BUSINESS_DATE`). | Read the `ErrorCode` / named token → react per sections 1–2. |

(Other mode numbers exist in the code; `1` and `4` are the two documented from the UAT incident — confirm the full enum by grepping `makeCashHold` in `Carbon-OMS/handler-core-api.go`.)

## Debugging a `holdCashFail`

1. **Find the mode.** `mode=1` → infra (endpoint down / wrong URL). `mode=4` → G3 core rejected.
2. **mode=1:** verify `G3SBApi_Url` (watch for the trailing-`"` quote bug → URL ends `/%22`), confirm host:port reachable. Escalate to DevOps + G3SB team.
3. **mode=4:** read the response — numeric `ErrorCode` → section 1; named token → section 2.
4. **`20001`/`20002`:** genuine insufficient balance — confirm account funding; not a bug.
5. **`ERROR_VALUE_DATE...`:** the OMS clock vs the G3 core business date diverged. Check `HCBusinessDateToSystemTime` (the canonical date table) — Carbon-OMS startup probe `[G3-DATE]` logs both. The core team must roll the UAT date; OMS `ENABLE_G3_DATE=true` is the defensive auto-fetch.
6. **Credentials:** if the SOAP request logs `UserID` / `UserPassword` that look like leftover placeholders, verify they are the correct per-environment creds.

## Quick-verify

- The numeric code table is from g3sb-api.md v1.09 (2019). Grep the OMS response-parsing structs (`RESULT_HOLD_XML`, `RESULT_CASH_RELEASE_XML`, `RESULT_ACCOUNT_CONTRACT` in `datastruct/g3b-api.go`) to confirm which fields the code actually reads.