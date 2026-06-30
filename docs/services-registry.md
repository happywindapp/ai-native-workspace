# Service Registry

> **Single source of truth** cho danh sách service. KHÔNG sửa bảng tay — sửa `services` trong
> `template.config.json` rồi chạy `pwsh scripts/apply-template.ps1`. Mọi doc khác tham chiếu file này.

## Services (theo tier)

<!-- BEGIN gen:services-registry -->
| Key | Service | Tier | Stack | Repo | Base URL env |
|-----|---------|------|-------|------|--------------|
| `web` | WebApp | frontend | React/TypeScript | `path/to/web` | `WEB_URL` |
| `admin` | AdminApp | frontend | React/TypeScript | `path/to/admin` | `ADMIN_URL` |
| `edge` | EdgeGateway | bff | Node/NestJS | `path/to/edge` | `EDGE_URL` |
| `orders` | OrderService | domain | Go | `path/to/orders` | `ORDERS_URL` |
| `billing` | BillingService | domain | Go | `path/to/billing` | `BILLING_URL` |
| `settlement` | SettlementService | domain | Go | `path/to/settlement` | `SETTLEMENT_URL` |
| `exchange-gw` | ExchangeGateway | integration | Go | `path/to/exchange-gw` | `EXCHANGE_GW_URL` |
<!-- END gen:services-registry -->

## Tier — ý nghĩa

| Tier | Vai trò |
|------|---------|
| `frontend` | Web/mobile app người dùng cuối |
| `bff` | Backend-for-frontend / API gateway / edge |
| `gateway` | Reverse proxy / ingress (nếu tách khỏi bff) |
| `domain` | Service nghiệp vụ (mỗi bounded context 1 service) |
| `integration` | Adapter hệ ngoài (bank, SOAP, message protocol…) |
| `infra` | CI/CD, observability, shared contracts (proto/OpenAPI) |

## Env theo tier

Xem [server-config.example.md](../server-config.example.md) (khối `gen:env`) — danh sách biến môi trường gom theo tier, cũng sinh từ cùng manifest.
