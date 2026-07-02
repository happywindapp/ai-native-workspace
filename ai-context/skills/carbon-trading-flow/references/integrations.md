# Carbon Integrations

Carbon-specific integration *business logic*. For SWIFT MT / FIX message *format* see `financial-messaging`; for G3 Core API see memory `g3-core-api-reference`; for VietinBank API see memory `vietinbank-api-integration`.

## Info Service — product list / detail

The Carbon market-data source for product symbols. Migrated to a new host 2026-05-07.

| | Old (BondOMS legacy) | New (Carbon) |
|---|---|---|
| Host | `http://192.168.82.87:8080` | `http://192.168.54.22:8084` |
| List | `GET /api/privateBond` | `GET /api/Carbon/products/list` |
| Detail | n/a | `GET /api/Carbon/products/{symbol}` |
| Env key | `URL_API_INFO_SERVICE` (same) | new value |

- MW service `Carbon-Middleware/src/services/handleResFromInfo.js` — `callToGetBond()` (list) + `callToGetBondDetail(symbol)` (detail).
- MW route: `GET /api/bonds`, `GET /api/bonds/{symbol}` (`controllers/bond.js` + `route/bond.js`, mounted at `/api`).
- FE calls via `getAPILink('bonds')` / `getAPILink('bonds/{symbol}')` (`services/index.ts`).

### Response shapes

**List** returns a flat array of strings: `["VN2026","HN2025","ACRBNEU24",...]`. MW transforms to `[{Symbol: code}]` to keep the FE contract.

**Detail** — the Carbon API drops Bond fields (`ParValue`, `IssuerName`, `IssueDate`, `MaturityDate`, `PeriodRemain`, `BondFeatures`, `SecurityTradingStatus`, `InterestRate*`). Mapping:

| Bond legacy | Carbon detail |
|---|---|
| `IssuerName` | `Issuer` |
| `IssueDate` | `ListingDate` |
| `MaturityDate` | `DeListingDate` |
| `PeriodRemain` | `RemainingTradingDays` |
| `SecurityTradingStatus` | `TradingStatus` |
| `BondFeatures` (multi-char) | `CarbonType` (single-char enum) |

Market-price fields are unchanged: `BasicPrice`, `Floor/CeilingPrice`, `Floor/CeilingPricePT`, `MatchPrice`, `ClosePrice`, `CurrentPrice`, `PTMatchPrice`, etc.

- **Mệnh giá (par value):** Carbon has no `ParValue`. The "Mệnh giá (VNĐ)" row displays `PTMatchPrice` (latest PT match price) — Carbon is a commodity with no fixed par value. `ReleaseStock.tsx` uses `bondItem?.PTMatchPrice ?? bondItem?.ParValue` (fallback for legacy Bond data).

### `CarbonType` enum (FIXED 2026-05-14 — was reversed)

Canonical source: InfoGate spec, FIX tag 167.

| Value | Meaning | Code length |
|---|---|---|
| `"1"` | **Tín chỉ** (credit) | 9 chars |
| `"2"` | **Hạn ngạch** (quota) | 6 chars |

Mapping FE `carbonTypeMapping` in `Carbon-Terminal/src/constant/index.js` was initially reversed (`1:Hạn ngạch / 2:Tín chỉ`) → e.g. VN2025 (`CarbonType:"2"`, 6-char = Hạn ngạch) displayed as "Tín chỉ". Fixed by swapping to match the InfoGate spec.

## HNX InfoGate — carbon market-data FIX messages

Source: `documents/Cac-bon - Dac ta InfoGate.pdf`. Two FIX messages added for the carbon pilot market. `BeginString = HNX.TDS.1`, direction OUT (HNX→member), no order entry. Session layer inherits the shared HNX.TDS protocol (Logon/Logout/Heartbeat/ResendRequest).

### MsgType `CBS` — "Cac-bon Info" (one product's detail)
Key new tags: **167 `Cac-bonType`** (`1`=Tín chỉ, `2`=Hạn ngạch — canonical), **5001 `RemainingTradingDays`**. Status tags 326/327/340 (see `product-domain-model.md`). Other tags mirror TPRL: prices (260 BasicPrice, 333 FloorPice, 332 CeilingPice, 31 MatchPrice, ...), PT (negotiated) qty/price pairs (393/3931, 395/3951, 396/3961, 381/3811, 382/3821), identifiers (55 Symbol, 425 BoardCode, 910 ISIN, 106 Issuer, 225 ListingDate, 541 DeListingDate).

### MsgType `CBB` — "Board Cac-bon Info" (board detail)
Key tags: 425 BoardCode, 426 BoardStatus, 336 TradingSessionID, 340 TradSesStatus, 270 TotalTrade, 250 TotalStock, 210/211 totalTradedQtty/Value, 240 totalPTTradedQtty, 3832/3833 PT_TotalBid/OfferValue, 341 MarketCode.

> **Naming note:** Carbon-Middleware / Info Service REST responses use PascalCase no-hyphen (`CarbonType`, `PTMatchPrice`); InfoGate FIX uses `Cac-bonType`, `PT_MatchPrice`. Same meaning, different spelling.

## HNX integration overview — 3 connections

| Connection | Protocol | Phase-1 status |
|---|---|---|
| **InfoGate** | FIX (`HNX.TDS.1`) — CBS/CBB messages | infrastructure reused, market-data feed |
| **InfoFile** | file-based trade results | infrastructure exists; Carbon format reuses HNX TDS CSV/XML |
| **Web Terminal** | standalone web app (username+password) | Phase 1 — manual order entry (dual entry) |
| Kênh trực tuyến (order-entry API) | — | Phase 2+ (no spec yet) |

**HNX Web Terminal** (`Car-bon - Terminal_ Tai lieu HDSD.pdf`): standalone web app, not embedded/iframe/SDK. Modules: Giao dịch (enter TTĐT/BCGD orders, pending book, KQGD book), Sổ lệnh quá khứ, Tra cứu. TTĐT order has an "Ẩn danh" checkbox; BCGD has no anonymous but adds counterpart member + counterpart account.

## Shared `bond-stp` service

The STP service is shared between Private Bond and Carbon — there is **no separate "carbon-stp" service**.

- Deployed as `bond-stp`, namespace `private-bond-uat`.
- Distinguishes project by URL path: `/v1/stps/carbon/*` vs `/v1/stps/private-bond/*`.
- Reads the shared `/g3sb-csd-receive/` folder and parses MT5xx for both projects; routing is at the URL/handler layer, not separate folders/processes.
- **Debugging Carbon STP:** read `bond-stp` logs in `private-bond-uat`, filtered by `/v1/stps/carbon` or carbon order IDs (`{YYYYMMDD}-TESTxx`). This is correct — do not look for a "carbon-stp" service.
- The MW `urlSTP` variable (`Carbon-Middleware/src/services/handleResFromSTP.js`) **must carry the `/v1/stps/carbon` prefix** — a missing or wrong (`private-bond`) prefix routes the callback to the Bond branch and the Carbon order/vsdStatus never updates.