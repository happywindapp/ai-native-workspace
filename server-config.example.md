# Server Config (EXAMPLE)

> Copy file này thành `server-config.md` (đã gitignore) rồi điền giá trị thật của bạn.
> TUYỆT ĐỐI KHÔNG commit `server-config.md`. File này chỉ liệt kê TÊN biến, không có giá trị.
>
> Bảng dưới sinh từ `template.config.json` (services + datastores + messaging) qua
> `pwsh scripts/apply-template.ps1` — đừng sửa tay phần giữa marker `gen:env`.

## Biến môi trường theo tier

<!-- BEGIN gen:env -->
**frontend**

| Service | Env var | Value |
|---------|---------|-------|
| WebApp | `WEB_URL` | `{{host}}:{{port}}` |
| AdminApp | `ADMIN_URL` | `{{host}}:{{port}}` |

**bff**

| Service | Env var | Value |
|---------|---------|-------|
| EdgeGateway | `EDGE_URL` | `{{host}}:{{port}}` |

**domain**

| Service | Env var | Value |
|---------|---------|-------|
| OrderService | `ORDERS_URL` | `{{host}}:{{port}}` |
| BillingService | `BILLING_URL` | `{{host}}:{{port}}` |
| SettlementService | `SETTLEMENT_URL` | `{{host}}:{{port}}` |

**integration**

| Service | Env var | Value |
|---------|---------|-------|
| ExchangeGateway | `EXCHANGE_GW_URL` | `{{host}}:{{port}}` |

**datastore**

| Store | Env vars |
|-------|----------|
| PostgreSQL | `PG_HOST` `PG_PORT` `PG_USER` `PG_PASSWORD` |
| Redis | `REDIS_HOST` `REDIS_PORT` `REDIS_USER` `REDIS_PASSWORD` |
| MongoDB | `MONGO_HOST` `MONGO_PORT` `MONGO_USER` `MONGO_PASSWORD` |

**messaging**

| Type | Env var |
|------|---------|
| Kafka | `MQ_URL` |
| gRPC | `GRPC_ENDPOINT` |
<!-- END gen:env -->

## Notes

- `{{host}}` / `{{port}}` là placeholder điền khi copy sang `server-config.md` thật.
- Mọi giá trị nhạy cảm sống trong `server-config.md` (gitignored) hoặc secret manager.
- Không hardcode credentials vào code/docs.
