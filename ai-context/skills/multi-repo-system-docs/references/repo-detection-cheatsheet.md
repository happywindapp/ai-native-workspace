# Repo Detection Cheatsheet

Map manifest/marker files to language, framework, and repo role. Read the manifest, not every file.

## Language / runtime markers

| Marker file | Language | Notes |
|---|---|---|
| `package.json` | Node.js / JS / TS | Check `dependencies` for framework |
| `go.mod` | Go | Module name = service identity |
| `*.csproj`, `*.sln` | C# / .NET | |
| `pom.xml`, `build.gradle` | Java / Kotlin | |
| `requirements.txt`, `pyproject.toml` | Python | |
| `Cargo.toml` | Rust | |
| `composer.json` | PHP | |

## Framework detection (from package.json deps)

| Dependency | Framework / role |
|---|---|
| `react`, `next` | Frontend (Next.js if `next`) |
| `vue`, `nuxt` | Frontend (Vue) |
| `@angular/core` | Frontend (Angular) |
| `express`, `koa`, `fastify` | Backend REST API |
| `@nestjs/core` | Backend (NestJS) |
| `socket.io`, `ws` | Realtime / WebSocket |
| `nats`, `kafkajs`, `amqplib` | Message-queue consumer/producer |
| `pg`, `mysql2`, `mongoose`, `typeorm`, `prisma`, `sequelize` | Database / ORM |

## Infra & integration markers

| File | Meaning |
|---|---|
| `Dockerfile` | Containerized service |
| `docker-compose.yml` | Multi-service local orchestration — read it for the full service map + ports |
| `azure-pipelines.yml`, `.github/workflows/` | CI/CD definition |
| `*.proto` | gRPC contract |
| `openapi.yaml`, `swagger.json` | REST API contract |
| `.env`, `*.env.*` | Environment config — read variable NAMES only, never values |
| `migrate-config.json`, `migrations/` | Database migrations |

## Repo role classification

- **Frontend**: has `react`/`vue`/`angular`, build outputs to `dist`/`build`/`.next`.
- **Backend API**: has HTTP framework + `route`/`controller`/`handler` folders.
- **Gateway**: integrates an external protocol (FIX, ISO8583, payment APIs) — folders like `gateway`, `fix`, `adapter`.
- **Shared library**: no entrypoint/server; imported by others (e.g. `*GoModules`, `*-common`, `*-shared`).
- **STP / processor**: consumes a queue, no direct user-facing API.

## Fast-scan order per repo

1. Manifest file → language + deps
2. `docker-compose.yml` (if any) → service topology + ports
3. Entrypoint (`main.go`, `index.js`, `Program.cs`, `app.py`)
4. `route`/`controller`/`handler` folder → API surface
5. `models`/`entities` + migrations → data model
6. `README.md` → stated intent (verify against code)