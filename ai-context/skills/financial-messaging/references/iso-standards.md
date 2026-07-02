# ISO 15022 vs ISO 20022, FIX vs SWIFT-MT

Background to place the HSC carbon messaging protocols in context.

## ISO 15022 — SWIFT FIN (MT messages)

The standard VSDC uses today. Tag-value text inside 5 FIN blocks (`{1:}{2:}{3:}{4:}{5:}`); block 3 optional. Message types are `MTnnn` (MT508, MT518, MT540…). Field formats use code words and qualifiers (`:22F::ACTP//QUOT`). Decades-old, proven, already deployed on the SWIFT network — hence still primary in Vietnam for depository/settlement.

See `swift-mt-tags.md` for the full block/tag detail.

## ISO 20022 — MX / XML (the successor)

XML-based, schema-validated (XSD). Message families:
- **camt** — cash management / account reporting (e.g. `camt.053` statement)
- **sese** — securities settlement (e.g. `sese.023` settlement instruction)
- **semt** — securities management / reporting (statements, corporate actions)

Advantages: schema validation catches errors early, extensible namespaces, standard XML/JSON tooling. In the carbon stack, the **FileAct `.par`** parameter file already references `Service=camt.xxx.fisp.rep` for EOD reports — a partial ISO 20022 touchpoint. Full MX migration is a future-phase item; FIX and MT are not deprecated near-term.

## FIX vs SWIFT-MT — when HSC uses which

| Aspect | FIX | SWIFT MT (ISO 15022) |
|---|---|---|
| Used for | Real-time orders, quotes, market data | Depository, settlement, account management |
| HSC channel | **HNX** — CBTS gateway + InfoGate | **VSDC** — `HSC_STP` `.fin` files |
| Transport | TCP/IP, direct, low-latency | File polling via VSDC Gateway Client |
| Encoding | ASCII tag=value, SOH-delimited | Tag-value in FIN blocks, code words |
| Latency | milliseconds | seconds–minutes (batch/polling) |
| Sequencing | session sequence numbers, ResendRequest | ISN/OSN per session, ACK/NAK |
| Auth | Logon message (user/pass) | BIC + (in full SWIFT) network auth; MAC/CHK trailer |
| Example messages | `35=D` NewOrder, `35=8` ExecutionReport | MT518 obligation, MT598.301 register |

## Carbon message pipeline — which protocol at each hop

```
Trader → Terminal            : Web UI (HTTPS/REST) — not a wire protocol
Terminal → Middleware        : HTTPS REST + JWT
Middleware → OMS / STP       : HTTP REST
HSC_STP ↔ VSDC               : SWIFT MT .fin files (ISO 15022) via folder polling
HSC ↔ HNX (orders, Phase 2)  : FIX 4.4 (CBTS gateway)
HNX → HSC (market data)      : FIX (InfoGate, BeginString HNX.TDS.1, CBS/CBB)
HNX → HSC (EOD results)      : InfoFile (CSV/XML)
```

Phase 1: VSDC settlement runs on SWIFT MT; HNX order entry is via the Web Terminal (manual), with InfoGate FIX feeding market data. FIX order-entry automation and ISO 20022 reporting are later-phase scope.

See `hnx-fix-gateway.md`, `hnx-infogate-terminal.md`, and the VSDC references for protocol detail.
