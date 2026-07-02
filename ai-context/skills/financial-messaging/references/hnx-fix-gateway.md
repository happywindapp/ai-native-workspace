# HNX FIX Gateway (CBTS)

FIX 4.4 protocol for HSC ↔ HNX order entry, quotes, and executions. Reference spec: `CBTS.FIX Message & Protocol for Gate v1.15` (TPDNRL bond gateway — carbon reuses the same structure). Phase 1 carbon order entry is via Web Terminal; this gateway is the protocol backbone.

## FIX message structure

```
8=FIX.4.4 | 9=<BodyLength> | 35=<MsgType> | 49=<Sender> | 56=<Target> | 34=<SeqNum> | 52=<SendingTime> | ...body... | 10=<CheckSum>
```
- Delimiter = SOH (ASCII `0x01`). Tag 8 first, tag 10 last.
- Tag 9 = byte count excluding tags 8/9/10. Tag 10 = 3-digit checksum (HNX does not validate in practice; `000` accepted).
- Tag 52 `SendingTime` = UTC `yyyyMMdd-HH:mm:ss`.

## Session layer (admin messages)

| MsgType (35=) | Name | Purpose |
|---|---|---|
| A | Logon | Client starts session; tag 108 `HeartBtInt` (15–100s, ~30) |
| 5 | Logout | End session |
| 0 | Heartbeat | Keep-alive; also sent by HNX immediately on order receipt |
| 1 | TestRequest | Ping; peer replies Heartbeat (tag 112 `TestReqID`) |
| 2 | ResendRequest | Gap fill — tag 7 begin, tag 16 end sequence |
| 4 | SequenceReset | Re-sync sequence after disconnect (tag 36 `NewSeqNo`) |
| 3 | Reject | Peer rejects invalid message |

**Session flow:** Logon → HNX validates format/credentials/sequence → Logon reply or Reject+close → exchange data → must send Heartbeat/TestRequest each `HeartBtInt`; no traffic for 2×`HeartBtInt` → auto-close.

**Sequence rules:** session messages use the current sequence; application messages use current+1. Resent messages carry tag 43 `PossDupFlag=Y` — do NOT dedup on sequence alone.

## Order entry (CTCK→HNX)

| MsgType | Name | Key tags |
|---|---|---|
| D | NewOrderSingle | 11 ClOrdID (unique per CTCK), 1 Account, 55 Symbol, 54 Side (1=buy,2=sell), 38 OrderQty, 40 OrdType, 44/640 Price, 369 |
| F | OrderCancelRequest | 11 new ClOrdID, 41 OrigClOrdID, 55 Symbol, 369 |
| G | OrderReplaceRequest | 11 new ClOrdID, 41 OrigClOrdID, 38 new Qty, 2238 OrgOrderQty, 640 new Price, 369 |

## Quote messages (TTĐT — electronic agreement trade)

| MsgType | Name | Key tags |
|---|---|---|
| S | Quote | 11 ClOrdID, 1 Account, 54 Side, 38 Qty, 640 Price, 513 RegistID (target member / 0=public), 1111 Is_Visible (1=anonymous), 64 SettlDate |
| R | QuoteRequest | 11 ClOrdID, 644 RFQReqID (HNX ref to quote), 640 Price, 38 Qty |
| AJ | QuoteResponse | 11 ClOrdID, 693 QuoteRespID, 694 QuoteRespType (1=accept), 2 CoAccount |
| Z | QuoteCancel | 11 ClOrdID, 644 RFQReqID |
| AI | QuoteStatusReport | 549 quote status (0=success,2=rejected,4=canceled,5=expired) |

## Execution & market status (HNX→CTCK)

| MsgType | Name | Key tags |
|---|---|---|
| 8 | ExecutionReport | 37 OrderID (HNX ref), 39 OrdStatus, 150 ExecType, 151 LeavesQty, 32 LastQty, 31 LastPx |
| h | TradingSessionStatus | 336 SessionID, 340 TradSesStatus |
| f | SecurityStatus | 55 Symbol, 326 SecurityTradingStatus (carries CBS/CBB payloads — see infogate ref) |
| g / e | TradSesStatusRequest / SecurityStatusRequest | request session / security state |

## Key tags & enums

| Tag | Field | Values |
|---|---|---|
| 11 | ClOrdID | Unique per CTCK — HNX rejects duplicates, even across sessions |
| 40 | OrdType | 2=LO, 3=MTL, 4=MAS, 5=ATC, 6=ATO, A=MAK, K=MOK, M=Market Maker, S=Quote |
| 39 | OrdStatus | 0=New,1=PartialFill,2=Filled,4=Canceled,8=Rejected,A=PendingCancel,E=PendingReplace |
| 150 | ExecType | 0=New,1=PartialFill,2=Fill,3=Done,4=Canceled,5=Replace,6=Reject,F=Trade |
| 326 | SecurityTradingStatus | 0=normal,2=halted,6=delisted,9=pending,10=suspended,11=restricted,25=special |
| 340 | TradSesStatus | 1=normal,2=halted,13=closed,90=await open,97=off-market |
| 369 | LastMsgSeqNumProcessed | Flow control — track buffer size |
| 1111 | Is_Visible | 1=anonymous (hide CTCK until execution), 0=public |
| 440 | Special_Type | Market-maker quote: 1=one-way, 2=two-way, 3=two-way w/ replace |

## Flow control & recovery

- **Buffer:** HNX caps pending (orders sent − confirmations) at **100**. Exceed → CTCK cannot send orders (session messages still allowed). On each order received HNX replies Heartbeat with updated tag 369.
- **Gap recovery:** detect missing sequence → ResendRequest; HNX redelivers with `PossDupFlag=Y`. SequenceReset only after major gap / reconnect.
- **Repos (Phase 2):** MsgTypes N01–N06 (inquiry/firm repos), MA/MB/MC/ME (BCGD multileg) — defined in spec, not active Phase 1.

See `hnx-infogate-terminal.md` for the InfoGate market-data feed and Web Terminal; `iso-standards.md` for FIX vs SWIFT-MT.
