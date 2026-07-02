# BondTerminal_FE Map — Frontend Structure

Next.js 13 Pages Router / React 18 / Redux-Saga / MUI v5 / MSAL (Azure AD). Trader UI. Calls BondTradingMiddleware + BondOMS.

> As-of: ĐTTTT Outright UI shipped 2026-05-12. Grep component names to confirm paths.

## Pages structure

Single tab-switcher pattern — there is NO standalone `/electronic-negotiated` route:

| Page | Tabs |
|---|---|
| `/home` | BCGD (ExecuteCommand) \| TTDT (PlaceQuoteCommand) |
| `/order-book` | BCGD (HeaderFilter + columns) \| TTDT (QuoteHeaderFilter + quoteColumns) |

## TTDT components

| File | Purpose |
|---|---|
| `src/components/container/Home/Components/PlaceQuoteCommand.tsx` | TTDT order entry form |
| `src/components/container/Home/Components/HomeTabSwitcher.tsx` | Pill-style tab switcher, shared by `/home` + `/order-book` |
| `src/components/container/OrderBook/Components/QuoteActions.tsx` | Accept/Cancel buttons per row |
| `src/components/container/OrderBook/Components/AcceptQuoteDialog.tsx` | Accept-quote confirmation dialog |
| `src/components/container/OrderBook/Components/QuoteHeaderFilter.tsx` | 8 filter fields, button-driven commit (applies only on "Tìm kiếm" click) |

## TTDT types & service

| File | Content |
|---|---|
| `src/components/container/ElectronicNegotiated/shared/types.ts` | `QuoteRecord`, `PlaceQuoteRequest`, `AmendQuoteRequest`, `CancelQuoteRequest`, `AcceptQuoteRequest`, `ListQuotesParams`. The `ElectronicNegotiated` folder name is a leftover after deleting the standalone route — only `shared/types.ts` remains. |
| `src/services/quote-service.ts` | `POST quotes/{add,edit,cancel,accept}` + `GET quotes/list`. Cancel uses POST (not DELETE) — consistent with add/edit/accept, avoids gateway stripping the body. |

## BCGD-side touch points (TTDT integration impact)

| File | Change |
|---|---|
| `ExecuteCommand.tsx` | added props `activeTab` + `setActiveTab` to render `HomeTabSwitcher` under title |
| `Home/index.tsx` | added `activeTab` state, conditional render BCGD vs TTDT, submit body `transactionType: "outright_bcgd"` |
| `OrderBook/index.tsx` | added "Loại GD" column (BCGD/ĐTTTT), tab switcher + tabpanels |
| `BondPlus/index.tsx` | added `transactionType: "outright_bcgd"` to submit body |
| `constant/index.ts` | added `transactionTypeMapper` (outright_ttdt → "ĐTTTT", outright_bcgd → "BCGD") |
| `interface/index.tsx` | added `transactionType?: string` to `IFromMOSRecords` |

## Patterns

- **Polling (TTDT):** `OrderBook/index.tsx` polls every 10s when `activeTab===1`, with a `quoteFetchInFlightRef` guard against overlapping requests on slow links.
- **Filter (TTDT):** button-driven, mirrors BCGD UX — draft state `quoteFilter` updates on keystroke; applied state `appliedQuoteFilter` commits only on "Tìm kiếm" click → `filteredQuoteRecords` filters on the applied state.
- **Counterparty detection:** `String(row.orderCompany).replace(/^0+/,"") !== String(codeHSC).replace(/^0+/,"")` — robust against "011" vs "11".
- **Account format validation:** synced with BCGD (length-10 check only); trusts BE format validation. `ACCOUNT_REGEX` not used on FE.

## 3-layer POST chain (cancel example)

```
FE  POST quotes/cancel
 → MW  POST /cancel            (route/quote.js:12)
 → MW→OMS  POST /equity/quote/cancel
 → OMS  quote.Post("/cancel", CancelQuoteHandler)
```

Deploy FE + MW together to avoid 404s.