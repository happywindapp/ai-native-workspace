# Carbon Account Rules

Account-field semantics and account-registration business rules.

## `accountTypeName` vs `investor_type` — two independent axes

The single most common Carbon bug source. They are **not** the same thing and must never be conflated.

| | `accountTypeName` | `investor_type` |
|---|---|---|
| Axis | Account type / product traded | Subject type (loại hình chủ thể) |
| Type | string `"quota"` / `"credit"` | integer `1` / `2` / `3` |
| MT mapping | `:22F::ACTP//QUOT\|CRDT` | `:22F::TPTY//EMIT\|PROJ\|ORGA` |
| Decides | which product-code length is valid (6↔quota / 9↔credit) | legal-entity classification of the investor |
| Used for code↔account validation? | **YES — only this** | **NO — never** |

**Why the name misleads:** `investor_type` (snake_case) reads like "investor classification" → tempting to treat it as a quota/credit proxy. It is actually the *subject type*. The name is kept for legacy reasons (it is woven across FE/MW/DB/STP/SWIFT — renaming is a high-risk cross-cutting change). **When you see `investor_type` in Carbon code, always read it as "subject type".**

**The 2026-05-19 bug:** `Home/index.tsx` constant `INVESTOR_TYPE_CREDIT = 2` treated `investor_type` as a quota/credit proxy → wrongly rejected credit accounts whose `investor_type ∈ {1,3}` (e.g. account `011C000002` = Cơ sở phát thải registered to buy credit) with "NĐT chưa đăng ký giao dịch loại sản phẩm này". Fixed by comparing `productType` vs `accountTypeName`.

### Cross-stack mapping — `accountTypeName`

| Layer | Location | Value |
|---|---|---|
| FE label | `Carbon-Terminal/src/constant/index.js` `typeOptions` | `"quota"` / `"credit"` |
| FE state — filter tab | AccountManagement/AccountUpdate/CustomersList | `customerType` |
| FE state — form modal | Register + edit modals | `investorType: string` (set from `row.accountTypeName`) |
| FE derive | `Home/index.tsx` `deriveProductType` | code length 6→`quota`, 9→`credit` |
| API payload | FE → MW `/accounts/add\|edit` | `accountTypeName` |
| MW persist | `account.model.js` | `Sequelize.STRING` nullable (no enum check) |
| DB | `accounts."accountTypeName"` | varchar nullable |
| MW → STP map | `handleResFromSTP.js` `actpTypeMap` | quota→`QUOT`, credit→`CRDT` |
| SWIFT | MT598.301 | `:22F::ACTP//QUOT\|CRDT` |

### Cross-stack mapping — `investor_type`

| Layer | Location | Value |
|---|---|---|
| FE label | `constant/index.js` `subjectTypeOptions` | `1`/`2`/`3` |
| FE rule | `Register/Components/InvestorType.tsx`, `CustomersList/Components/ContentDetail.tsx` | quota→fixed 1 readonly; credit→choose 1\|2\|3 |
| API payload | FE → MW | `investor_type: number` |
| MW persist | `account.model.js` | `Sequelize.INTEGER` nullable (no enum check) |
| DB | `accounts.investor_type` | int nullable |
| MW → STP map | `handleResFromSTP.js` `tptyTypeMap` | 1→`EMIT`, 2→`PROJ`, 3→`ORGA` |
| STP handler | `HSC_STP/handler/model/register_account.go` | string field, forward-only |

### Validity matrix `(accountTypeName, investor_type)`

| `accountTypeName` | Allowed `investor_type` | Note |
|---|---|---|
| `quota` | `1` only | UI forces readonly = 1 |
| `credit` | `1` ✅ / `2` ✅ / `3` ✅ | account `011C000002` is credit + investor_type 1 |

`quota+2` / `quota+3` are invalid (UI readonly blocks them). MW does **not** enforce this server-side.

## Permanent registration — no `endDate`

- Carbon trading accounts are registered once and used forever → **no end date**.
- FE does not display or send `endDate`; MW does not validate `endDate <= startDate`; OMS has zero `endDate` reference.
- DB column `endDate` is kept (null) for legacy compatibility; STP forwards `end_date: null`, Go service skips processing when empty.
- The 2026-05-13/14 cleanup removed `endDate` from FE Register/CustomersList (state, payload, UI, filter, export), MW validation `end_date_invalid`, and a dangerous Bond-inherited side-effect in `account.js getList` that auto-nullified `accountTypeName` for accounts with `endDate < now`.
- **If you see `endDate` reappear in a new payload or a `endDate <= startDate` validation → suspect leftover Bond code, propose removal, but keep the DB column.**

## Subject type fixed by account type

- **quota account** → `investor_type = 1` ("Cơ sở phát thải") only, not selectable → UI renders a readonly TextField.
- **credit account** → dropdown of 3 options (1 Cơ sở phát thải / 2 Chủ dự án tín chỉ / 3 Tổ chức khác).
- Applies to Register and the CustomersList edit modal. AccountManagement/AccountUpdate keep `investorType` state but do not render this dropdown.

## Single payment method

- Carbon has one payment method: "Thanh toán ngay" (`paymentType = 1`, T+0).
- FE renders a readonly TextField instead of a Select. No end-of-day settlement (that was Bond Plus, removed).

## Doctype mapping chain — `95S::ALTE//VISD/<code>/<country>/<id>`

Four vocabularies for the same "document type" concept; applies to Carbon register + update.

| G3 num | G3 camelCase | FE docType (snake_case) | STP idType (camelCase) | MT 4-char | Vietnamese |
|---|---|---|---|---|---|
| `1` | `socialId` | `id_card` | `socialId` | `IDNO` | Chứng minh thư |
| `2` | `passport` | `passport` | `passport` | `CCPT` | Hộ chiếu |
| `4` | `otherId` | `other_certificate` | `otherId` | `OTHR` | Chứng thư khác |
| `5` | `businessId` | `business_license` | `businessId` | `CORP` | Giấy phép kinh doanh |
| `10` | `gov` | `government_agency` | `gov` | `GOVT` | Cơ quan chính phủ |
| `11` | `tradingCodeForeRetail` | `trading_code_individual_foreign` | `businessIdForInd` | `ARNU` | Trading Code cá nhân NN |
| `12` | `tradingCodeForeIns` | `trading_code_org_foreign` | `businessIdForCorp` | `FIIN` | Trading Code tổ chức NN |

- **G3 ⟷ STP mismatch** for the last 2 rows: G3 uses `tradingCodeFore*`, STP uses `businessIdFor*`. FE snake_case is the canonical middle layer — G3→FE→MW→STP, each pair has its own map, so the flow doesn't break as long as the 3 maps stay in sync.
- Where each map lives: FE `Carbon-Terminal/src/constant/index.js` `documentTypeOptions`; MW `Carbon-Middleware/src/constants/index.js` `docTypeFEToSTP` + `mapDocTypeFEToSTP()`; STP `HSC_STP/internal/domain/hsc_stp/model/register_account.go` `typeRegisterByIdType` (active only when `fromService == "carbon"`).
- G3 returns `accountInfo.IDType` already in STP camelCase (`socialId`/`passport`/...), not Bond numeric. FE maps back to snake_case via `mapG3IDTypeToFEDocType` to prefill the Register form. 3 cases (`gov`/`businessIdForCorp`/`businessIdForInd`) G3 does not return → user must pick from the dropdown.
- STP keeps two parallel maps: `typeRegisterByMemberType` (Bond legacy, key = numeric `memberType`) and `typeRegisterByIdType` (Carbon, key = camelCase `idType`); switched by `fromService` — editing Carbon does not break Bond.