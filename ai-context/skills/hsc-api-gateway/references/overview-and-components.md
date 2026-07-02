# Overview, Components & Deployment Environments

## Project purpose

`c:\_core_api_gateway` is the API gateway layer for **HSC (Ho Chi Minh Securities)** — a Vietnamese securities/trading platform. It is the central orchestration layer between client apps and multiple backend financial systems (equity, derivatives, OTC, risk control, banking).

**Why it exists:** HSC has a legacy XML/SOAP trading core ("G2") plus newer SQL Server systems (G3SB equity, G3FB derivatives, RiskControl). The gateway translates modern REST/JSON to/from the legacy systems and aggregates data from multiple sources.

**Domain assumptions when working here:**
- Domain is **Vietnamese stock trading** (HNX, HOSE, UPCOM markets) + **futures/derivatives** + bond/OTC.
- Timezone is hardcoded `Asia/Bangkok` in many places (= Vietnam time) — used for trading-day cutoffs.
- Currency is VND, no internationalization. Amounts often parsed as comma-separated strings.
- **Equity** (`/equity/*`) and **Derivatives** (`/derivatives/*`) are kept separate end-to-end — handlers, functions, datastructs, and DBs duplicated per asset class on purpose.
- "KRX" in deployment refers to the **Korea Exchange** migration — HSC is migrating Vietnam exchange tech to a KRX-based core.

## Component / folder map

| Folder | Role |
|---|---|
| `CoreApiGateway/` | **Main service.** Go 1.23 + Fiber v2, ~13.7k LOC, port 3030. Entry `main.go`, routes `router.go`. Pattern: `handler-api-*.go` → `func-*.go` → external systems. |
| `XmlAutoLogin/` | Tiny standalone Go cron service (~140 LOC). Default cron `0 0 5 * * *` (5 AM Asia/Bangkok). Logs into legacy XML API per endpoint in `JOB_XML_LIST` as a daily keepalive / smoke-test. Does NOT write the gateway's Redis session. |
| `CoreApiGateway-deployment/` | Kustomize K8s manifests. `core-api/base/` has 4 workloads; overlays per environment. Images → `hscitcontainerrepo.azurecr.io`. |
| `PnL-Deployment/` | Kustomize manifests only — PnL **service code is NOT in this repo**. |
| `Portfolio-Deployment/` | Kustomize manifests only — Portfolio service code is NOT in this repo. |
| `one-trading-core-api/` | Empty placeholder / TODO. Likely future unified equity+derivatives API. |
| `poc/` | Proof-of-concept mirroring CoreApiGateway structure. NOT deployed — treat as scratch. |
| `docs/` | API documentation. `Core API Gateway.postman_collection.json` (89 endpoints, runnable) + `Core Api Gateway APIs.xlsx` (canonical spec, 5 sheets: Equity / Derivatives / Common / OTP / E-Invoices). |

**Naming traps:** "the gateway" = `CoreApiGateway`; "auto-login" = `XmlAutoLogin`. Deployment folders are deceptively named — they only hold K8s YAML. If asked about PnL or Portfolio service *code*, it lives in a different repo.

## CoreApiGateway datastruct layout

- `datastruct/api-func.go` — REST DTOs (request/response bodies).
- `datastruct/core.go` — XML/SOAP types for the legacy G2 core.
- `datastruct/common.go` — cached domain models.

## Deployment environments

`CoreApiGateway-deployment/core-api/overlays/` Kustomize overlays:

| Overlay | Represents |
|---|---|
| `prod/` | Live production on the legacy Vietnam exchange backend |
| `uat/` | UAT for the legacy backend |
| `krx-prod/` | Production pointing at the new **KRX (Korea Exchange) core** |
| `krx-uat/` | UAT for the KRX backend |
| `trading/` | Internal/proprietary trading desk environment |

**KRX context:** Vietnam's national exchange is migrating its matching engine and clearing tech to a KRX-based stack. HSC maintains separate `krx-*` overlays so the same gateway image can target either backend. Config differences between `uat/` and `krx-uat/` are intentional, not drift.

**Base workloads** (`core-api/base/`):
- `core-api-gw` — main API
- `core-api-gw-order` — order-processing variant or sidecar (confirm before assuming)
- `xml-auto-login` — session keepalive
- `kong-plugin` — correlation-id plugin for the Kong API Gateway fronting the service

**Deployment conventions:**
- Image registry `hscitcontainerrepo.azurecr.io` (Azure Container Registry); CI is Azure Pipelines (`azure-pipelines.yml`).
- New env var pattern: declare in `base/` deployment, override per-env via the overlay's ConfigMap/Secret patch.
- Before editing an overlay, check whether the change belongs in `base/` instead — overlay-only edits drift across environments.