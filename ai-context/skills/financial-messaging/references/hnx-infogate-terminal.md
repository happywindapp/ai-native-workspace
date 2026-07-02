# HNX InfoGate (Market Data) & Web Terminal

The two HNX channels live in Phase 1 alongside the FIX order gateway.

## InfoGate — market-data FIX feed

- **Direction:** OUT only (HNX → trading members). No order entry.
- **BeginString:** `HNX.TDS.1`. Standard FIX 4.4 session layer (Logon/Heartbeat/ResendRequest) + custom MsgTypes `CBS` / `CBB`.
- **Delivery:** event-driven (price/status change) + periodic snapshots. Gap recovery via ResendRequest. Dissemination-only — no application rejects.

### CBS — Carbon Product Info
Sent when a product symbol changes, price updates, or a new listing occurs.

| Tag | Field | Notes |
|---|---|---|
| 55 | Symbol | Carbon product code (e.g. `VN2025`, `CHTAN3D26`) |
| 425 | BoardCode | Trading board |
| **167** | **CarbonType** | **1 = Tín chỉ (credit), 2 = Hạn ngạch (quota)** |
| **5001** | **RemainingTradingDays** | **Carbon-specific — countdown to delisting** |
| 326 | TradingStatus | 0=halted, 1=normal, 2=special |
| 327 | ListingStatus | 0=listed, 1=not yet, 2=new listing, 3=delisted |
| 336 / 340 | TradingSessionID / TradSesStatus | 340: 1=accept orders, 2=halted, 13=close, 90=await, 97=off |
| 225 / 541 | ListingDate / DeListingDate | yyyyMMdd |
| 910 / 106 | ISINCode / Issuer | issuer e.g. MARD for credit |
| 109 | TotalListingQtty | listed quantity (ton CO2) |
| 260 / 332 / 333 | BasicPrice / CeilingPrice / FloorPrice | reference / trần / sàn |
| 432 / 433 | CeilingPricePT / FloorPricePT | trần/sàn for PT (thỏa thuận) trades |
| 31 / 32 | MatchPrice / MatchQtty | last match |
| 139 / 266 / 2661 | ClosePrice / HighestPrice / LowestPrice | day close/high/low |
| 387 / 3871 | TotalVolumeTraded / TotalValueTraded | day totals |
| 391 / 392 | NM_TotalTradedQtty / Value | normal (khớp lệnh) totals |
| 393/3931, 395/3951, 396/3961, 394/3941 | PT_Match/Max/Min/Total Qtty/Price | agreement-trade stats |
| 381/3811, 382/3821 | PT_BestBid/BestOffer Qtty/Price | order-book best |
| 3301 | RemainForeignQtty | foreign room remaining |

### CBB — Carbon Board Info
Sent on session open, board-status change, or roll-up.

| Tag | Field |
|---|---|
| 425 / 426 | BoardCode / BoardStatus |
| 336 / 340 | TradingSessionID / TradSesStatus |
| 270 / 250 | TotalTrade / TotalStock |
| 210/211 | TotalTradedQtty / Value |
| 220/221, 240/241 | Normal vs PT (thỏa thuận) traded Qtty/Value |
| 3832 / 3833 | PT_TotalBidValue / PT_TotalOfferValue |
| 341 | MarketCode |

**Known caveats:** spec text shows typos `FloorPice`/`CeilingPice`/`ClosePice` (missing `r`) — verify against the live wire. Tag 167 collides with standard FIX `SecurityType` — confirm HNX override.

## Web Terminal — Phase 1 manual order entry

Standalone web app (no iframe/SDK). Auth: username + password (8–20 chars, ≥3 of 4 char classes, no spaces, must change on first login).

### TTĐT — Thỏa thuận điện tử (electronic agreement)
One party quotes; counterparty accepts from the order book. Fields: Mua/Bán, Mã SP (filtered to tradable status), Ngày GD (today, readonly), Khối lượng (positive int), Giá (≤13 digits, within trần/sàn), PT thanh toán ("Thanh toán ngay", readonly), Giá trị GD (auto = KL×Giá), **Ẩn danh** (checkbox → FIX tag 1111=1), Thành viên nhận (1 member / group / Public default).

### BCGD — Báo cáo giao dịch (trade report)
Two parties report an OTC trade. Differences from TTĐT: **no Ẩn danh**; has **TV đối ứng** (mandatory single) + **STK đối ứng** (shown only for internal same-member trades).

### Account code (STK) format
`[3-char member depository code][1 letter ∈ C|P|F|E|A|B][up to 6 digits]` — total 4–10 chars. First 3 chars own the prefix (no duplicates across members). The 4th-char semantics are undocumented — verify with member depository ops.

### Order books
- **Sổ lệnh chờ** — tabs Đã gửi (edit / cancel / treo) and Đã nhận (execute: enter STK mua/bán → confirm).
- **Sổ lệnh KQGD** — same-day matched/reported trades; printable.
- **Sổ lệnh quá khứ** — historical TTĐT/BCGD/KQGD, requires date range.

## InfoFile
File-based EOD trade-result delivery (CSV/XML, reuses HNX TDS format). Infrastructure already in place; carbon-specific format not yet detailed.

See `hnx-fix-gateway.md` for the order-entry FIX protocol, `iso-standards.md` for protocol comparison.