# Bond HNX FIX Dialect (HnxQuickfix)

FIX 4.4 dialect dùng cho **HSC Bond Trading** ↔ HNX. Khác Carbon ở chỗ Bond dùng **Cross orders** (s/t/u) và **NQuote/Multileg** thay vì NewOrderSingle (D/F/G).

Authoritative spec: `c:\_project_git\HnxQuickfix\spec\FIX44.xml` (6920 lines, HSC fork of upstream quickfixgo v0.7.0).

Module: `dev.azure.com/HSC-Inhouse-Dev/PrivateBond/_git/HnxQuickfix.git` (go 1.20).

## Services topology

| Service | Role | Lang |
|---|---|---|
| BondTerminal_FE | UI | React/TS |
| BondTradingMiddleware | API gateway + routing | Node.js |
| BondOMS | Order management | Go |
| **BondFIXOrderGW** | FIX 4.4 client (initiator) → HNX | Go (uses HnxQuickfix) |
| HSC_STP | MT messaging → VSD | Go |

## Message types — Bond Outright (CROSS, không phải NewOrderSingle)

| msgtype | Name | Hướng | Phân biệt |
|---|---|---|---|
| `s` | NewOrderCross | us→HNX (PLACE) | Có 38(qty), 640(Price2), 6363(SettlMethod), 6464(SettlValue), 64(SettlDate) |
| `t` | CrossOrderCancelReplaceRequest | us→HNX (**SỬA**) | Có qty/price MỚI + 41 OrigClOrdID + 551 OrigCrossID |
| `u` | CrossOrderCancelRequest | us→HNX (**HỦY**) | KHÔNG có qty/price, chỉ 11/55/54/549/551 |
| `8` | ExecutionReport | HNX→us | 150=0(New)/4(Cancelled)/5(Replaced)/F(Filled), 39 OrdStatus |
| `3` | Reject | HNX→us | 372 msgtype-rejected, 373 reason (negative int, vd -32001 cho `s`, -34000 cho `u`) |

⚠️ **`t` = SỬA, `u` = HỦY** (đây là FIX 4.4 standard, nhưng dễ nhầm với Carbon nơi dùng D/F/G).

## Message types — ĐTTTT Outright

| msgtype | Name | QuoteType 537 |
|---|---|---|
| `R` | QuoteRequest | — |
| `S` | Quote | — |
| `Z` | QuoteCancel | — |
| `AI` | QuoteStatusReport | 1-6 |
| `AJ` | QuoteResponse | — |

## Message types — Repo (HNX custom, KHÔNG có standard FIX)

| msgtype | Name | Mục đích |
|---|---|---|
| `MA` | NMultilegOrder | Place repo multi-leg |
| `ME` | NMultilegOrderReplaceRequest | Sửa repo |
| `MC` | NMultilegOrderCancelRequest | Hủy repo |
| `MR` | NMultilegOrderReplaceRequest **+** ReportNMultilegOrder | Đa nghĩa (theo direction) |
| `EE` | ExecOrderRepos | Báo cáo execution repo |
| `N01` | NQuote | ĐTTTT Repo new |
| `N02` | NQuoteStatusReport | ĐTTTT Repo status |
| `N03` | NQuoteRequest | ĐTTTT Repo request |
| `N04` | NQuoteStatusReportFirm | ĐTTTT Repo firm status |
| `N05` | NQuoteResponse | ĐTTTT Repo response |

## Reference data + Session

| msgtype | Name | Hướng |
|---|---|---|
| `A` | Logon | bidi (553/554 user/pass REQUIRED) |
| `0` | Heartbeat | bidi (HeartBtInt=30) |
| `e` | SecurityStatusRequest | us→HNX (boot-time) |
| `f` | SecurityStatus | HNX→us (bond master snapshot, ~938 msgs lúc logon) |
| `h` | TradingSessionStatus | HNX→us (phase broadcast mỗi 30s) |

## Header customization

**Tag 369 LastMsgSeqNumProcessed REQUIRED** trong header (FIX 4.4 std là optional). Mọi message Bond phải có 369.

Setting config: `EnableLastMsgSeqNumProcessed=Y`.

## Critical custom tags

| Tag | Name | Note |
|---|---|---|
| 109 | TotalListingQtty | HNX rename (std=ClientID) |
| 334 | **Parvalue** (INT enum) | HNX rename, enum 1=CANCEL, 2=ERROR, 3=CORRECTION |
| 369 | LastMsgSeqNumProcessed | Header REQUIRED |
| 537 | QuoteType | enum 1-6 (1=NEW, 2=REPLACE, 3=CANCEL, 4=CLOSE, 5=DONE, **6=PARTNERCANCEL**) |
| 549 | CrossType | enum 1-6 (1=AON, 2=IOC, 3=ONE_SIDE, 4=SAME_PRICE, **5=HNXACCEPT, 6=HNXREJECT** HNX-specific cross-firm) |
| 553/554 | Username/Password | Logon auth |
| 640 | **Price2** | HNX dùng tag 640 thay vì tag 44 cho price |
| 3321/3322 | HighPxOut, HighPxRep | Custom outright/repo high |
| 3331/3332 | LowPxOut, LowPxRep | Custom outright/repo low |
| 4488 | **OrderPartyID** | Firm ID, dùng Quote/Multileg |
| 4499 | InquiryMember | Custom |
| 6251 | TypeRule | Segment code (vd PCBOND_BRD_01) |
| 6363 | **SettlMethod** (FLOAT) | enum 1=IMMEDIATELY, 2=END_OF_DAY |
| 6464 | **SettlValue** (AMT) | Settlement cash value |
| 6465 | SettlValue2 | — |
| 9735/9736 | Allowed_Trading_Subject (Buy/Sell) | — |
| 9740-9745 | Price thresholds Normal/OutRight/Repo × upper/lower | — |
| 5632 | RepoMatchType (INT enum) | 1=REPOLEG1, 2=REPOLEG2 |
| 2260 | HedgeRate (FLOAT) | Repo |
| 2261 | ReposInterest (FLOAT) | Repo |

## NewOrderCross (35=s) required fields

Theo spec line 1590:

```
CrossID, CrossType, Side, ClOrdID, OrdType, Account, OrderQty,
EffectiveTime, Symbol, CoAccount, PartyID, CoPartyID,
Price2 (640), SettlValue (6464), SettlMethod (6363), SettlDate
```

## Session lifecycle (HnxQuickfix Application interface)

```go
type Application interface {
    OnCreate(SessionID)
    OnLogon(SessionID)
    OnLogout(SessionID)
    ToAdmin(*Message, SessionID)
    ToApp(*Message, SessionID) error
    FromAdmin(*Message, SessionID) MessageRejectError
    FromApp(*Message, SessionID) MessageRejectError
}
```

Acceptor: `quickfix.NewAcceptor(app, storeFactory, settings, logFactory)` — config `ConnectionType=acceptor`, `SocketAcceptPort=1369`.

Initiator (BondFIXOrderGW production): `quickfix.NewInitiator(...)` — config `SocketConnectHost=192.168.212.196:1369`.

## Reference message inventory (log production 2026-04-24, 4853 lines)

| 35= | Count | Pct |
|---|---|---|
| h (TradingSessionStatus) | 2709 | 56% |
| 0 (Heartbeat) | 1124 | 23% |
| f (SecurityStatus snapshot) | 938 | 19% |
| 8 (ExecutionReport) | 28 | <1% |
| s (NewOrderCross — place) | 26 | <1% |
| t (Cancel/Replace — sửa) | 17 | <1% |
| u (Cancel — hủy) | 4 | <1% |
| 3 (Reject) | 2 | <1% |
| AI (QuoteStatusReport) | 2 | <1% |
| A (Logon) | 2 | <1% |
| e (SecurityStatusRequest) | 1 | <1% |

→ ~98% volume là session/reference data broadcast.

## HNX Simulator project

Service Go giả lập HNX cho test khi HNX thật down — placeholder repo tại `c:\_project_git\HNXSimulator\`. Implement `quickfix.Application` interface + Acceptor mode + replay reference data + scenario REST API (inject reject/halt/disconnect).

Decision rationale + 5 open scope questions trong project memory (`bond-hnx-simulator-decision.md`).

## So sánh Bond vs Carbon FIX dialect

| | Bond (HNX) | Carbon (HNX) |
|---|---|---|
| Spec | `HnxQuickfix/spec/FIX44.xml` | `CBTS.FIX v1.15` + carbon extensions |
| Service | BondFIXOrderGW | (Phase 1: Web Terminal, no FIX yet) |
| Order msg | **s/t/u (Cross)** | D/F/G (Single) |
| Quote msg | R/S/AI/AJ/Z + N01-N05 (Repo) | R/S/AI/AJ/Z |
| Custom tag | 4488, 6363, 6464, 640 | 167 (CarbonType), 5001 (RemainingTradingDays) |
| InfoGate | Không dùng | Có (BeginString=HNX.TDS.1) |
