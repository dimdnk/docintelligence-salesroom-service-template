# Document Intelligence Digital Sales Room 📇

A multi-tenant, JWT-secured microservice on the iQ Key Value platform, backed by PostgreSQL, RabbitMQ, MinIO, and Ollama.

## About

The Digital Sales Room powers the **Digital Sales Room for Events** — the client-facing salesroom your buyers can actually use. Select venues from your library, generate a branded interactive page in one click, and send the link. Your client browses, asks questions, and approves — no PDF, no account, no back-and-forth.

- **Brief to pitch in minutes** — pick venues, click Generate, send a link. Done before the client follows up.
- **No client login required** — the client opens a link on any device, browses photos and floor plans, and approves.
- **Questions in context** — the client comments on a specific venue; you answer right there, not scattered across email threads.
- **Immutable approval record** — on approval, every field, file, and decision is locked with a timestamp. No disputes later.
- **Who it's for** — event agencies pitching venue shortlists to their clients.

The domain slices above are still to be built; this repository currently ships only the platform plumbing and the `ping` reference slice (see [Planned surfaces](#planned-surfaces)).

### What this starter includes

- **Venue library access** — reads venues from the shared catalog database via the `com.iqkv:docintelligence-venue` data layer already on the classpath; this service owns pitches, not the venue library
- **Feature-sliced layout** — one package per vertical feature slice; `ping` ships as the reference slice
- **Three security tiers** — public, tenant JWT (`MEMBER` / `TENANT_OWNER` / `ADMIN`), and platform admin (`PLATFORM_ADMIN`)
- **Multi-tenant isolation** — tenant context resolved from the `X-Tenant-ID` header or the JWT `tenant_id` claim before every request
- **Persistence** — PostgreSQL via MyBatis (XML mappers); the same database as `docintelligence-catalog-service` in production; Liquibase manages system and per-tenant schema migrations
- **Object storage** — MinIO (S3-compatible) client bound to a private `salesroom-assets` bucket; clients receive presigned URLs
- **Messaging** — RabbitMQ consumer for IAM tenant provisioning events (`tenant.created` / `tenant.updated`)
- **AI connectivity** — Spring AI Ollama starter pre-wired via `spring.ai.ollama.*` (base URL and models configurable per environment)
- **Observability** — Micrometer + Prometheus + structured JSON logging (Logstash encoder)
- **Quality tooling** — Checkstyle, JaCoCo, ArchUnit, Husky git hooks, commitlint

## Quick Links

- [API Documentation](./docs/api/README.md)
- [Architecture Overview](./docs/architecture/README.md)
- [Deployment Guide](./docs/deployment/README.md)
- [Contributing Guidelines](.github/CONTRIBUTING.md)

## API

Base path: `/api/v1/salesroom`

### Ping

| Method | Path           | Auth                 | Description                                                           |
| :----- | :------------- | :------------------- | :-------------------------------------------------------------------- |
| `GET`  | `/ping`        | public               | Unauthenticated reachability probe — no JWT or tenant header required |
| `GET`  | `/tenant/ping` | JWT + `X-Tenant-ID`  | Tenant-scoped probe — echoes the resolved tenant key                  |
| `GET`  | `/admin/ping`  | JWT `PLATFORM_ADMIN` | Admin probe — echoes the caller's `user_id` claim                     |

### Planned surfaces

`ping` is the only shipped surface — no domain endpoints exist yet. The slices below are the ones the Digital Sales Room brief calls for:

- **Venue selection** — search the venue library and add venues to a pitch
- **Sales Room generation** — one click renders the selected venues as a branded, interactive client page
- **Share links** — client-scoped links that open on any device with no account
- **Client engagement** — photo and floor-plan browsing, plus per-venue comments and Q&A answered in context
- **Approval** — client approval locks every field, file, and decision into an immutable, timestamped record

## Tech Stack

- Java 25 / Spring Boot (latest via parent POM)
- Spring AI Ollama starter (local LLM connectivity)
- MyBatis 3.x (no JPA) + PostgreSQL 17
- Liquibase for schema migrations
- RabbitMQ for async messaging
- Spring Security + OAuth2 Resource Server (RS256 JWT)
- MinIO client for S3-compatible object storage
- Micrometer + Prometheus
- springdoc-openapi (Swagger UI)

## Observability

- **Actuator endpoints** — `health` (liveness + readiness probes), `info`, `metrics`, and `prometheus` exposed on the management port (`8081`)
- **Structured logging** — JSON log output via the Logstash encoder; every request carries a correlation ID set by `CorrelationIdFilter`
- **Metrics** — no service-specific custom meters yet; the Micrometer/Prometheus registry exposes the standard JVM, HTTP, and Spring Boot meters

## Prerequisites

- JDK 25 (Eclipse Temurin)
- Maven 3.9+
- Node.js >= 22.15.0 & pnpm >= 10.33.2 (git hooks)
- Docker & Docker Compose
- [Ollama](https://ollama.com) running locally with at least one model pulled (e.g. `ollama pull llama3.2`)

## Quick Start

```bash
# Clone the repository
git clone https://github.com/IQKV/docintelligence-salesroom-service.git
cd docintelligence-salesroom-service

# Install git hooks
pnpm install

# Pull a model into Ollama (one-time)
ollama pull llama3.2

# Copy environment variables
cp .env.example .env.local
# Defaults point to Ollama on localhost:11434 — no edits needed for first-run

# Start infrastructure dependencies (PostgreSQL, RabbitMQ, MinIO)
docker compose up -d

# Run the service
./mvnw spring-boot:run -Pdev
# → API:          http://localhost:8080
# → Actuator:     http://localhost:8081/actuator/health
# → Swagger:      http://localhost:8080/swagger-ui.html
# → MinIO:        http://localhost:9003 (console)
```

## Environment Variables

| Variable                              | Default                  | Description                                               |
| :------------------------------------ | :----------------------- | :-------------------------------------------------------- |
| `DB_HOST`                             | `localhost`              | PostgreSQL host                                           |
| `DB_PORT`                             | `5432`                   | PostgreSQL port                                           |
| `DB_NAME`                             | `salesroomservice`       | Database name (the shared catalog database in production) |
| `DB_USERNAME`                         | `svc_salesroom_dba`      | Database user                                             |
| `DB_PASSWORD`                         | `svc_salesroom_dba`      | Database password                                         |
| `RABBITMQ_HOST`                       | `localhost`              | RabbitMQ host                                             |
| `RABBITMQ_PORT`                       | `5672`                   | RabbitMQ AMQP port                                        |
| `RABBITMQ_USERNAME`                   | `svc_salesroom_rmq`      | RabbitMQ user                                             |
| `RABBITMQ_PASSWORD`                   | `svc_salesroom_rmq`      | RabbitMQ password                                         |
| `OBJECTSTORAGE_ENDPOINT`              | `http://localhost:9000`  | S3-compatible endpoint (MinIO in SIT/UAT)                 |
| `OBJECTSTORAGE_PUBLIC_ENDPOINT`       | _(empty)_                | Public URL used to rewrite presigned URLs                 |
| `OBJECTSTORAGE_ACCESS_KEY`            | `iqkv`                   | S3 access key                                             |
| `OBJECTSTORAGE_SECRET_KEY`            | `iqkv_password`          | S3 secret key                                             |
| `OBJECTSTORAGE_BUCKET_NAME`           | `salesroom-assets`       | Private bucket holding salesroom assets                   |
| `OBJECTSTORAGE_REGION`                | `us-east-1`              | S3 region                                                 |
| `OBJECTSTORAGE_SSL`                   | `false`                  | Use HTTPS for the S3 endpoint                             |
| `SPRING_PROFILES_ACTIVE`              | `local`                  | Active Spring profile                                     |
| `MESSAGING_ENABLED`                   | `false`                  | Maps to `iqkv.messaging.rabbitmq.enabled`                 |
| `SPRING_AI_OLLAMA_BASE_URL`           | `http://localhost:11434` | Ollama server URL                                         |
| `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL` | `llama3.2`               | Model to use (must be pulled in Ollama first)             |
| `ROLLOUT_MODE`                        | `MULTI_TENANT`           | Platform rollout mode                                     |

> Copy `.env.example` to `.env.local` / `.env.uat` / `.env.prd` and fill in values per environment. The defaults point to a local Ollama instance — no API key needed for local development.
> The local Compose stack publishes MinIO on host port `9002` (console on `9003`) — set `OBJECTSTORAGE_ENDPOINT=http://localhost:9002` when running the service outside Docker.

## Maven Commands

```bash
# Build and test (skip Checkstyle during development)
./mvnw clean verify -Dcheckstyle.skip=true

# Run tests only
./mvnw test -Dcheckstyle.skip=true

# Explicit Checkstyle check
./mvnw checkstyle:check

# Coverage report → target/site/jacoco/index.html
./mvnw jacoco:report

# Production build
./mvnw clean package -Pproduction
```

## Docker

```bash
# Build image
docker build -t iqkv/docintelligence-salesroom-service:latest .

# Run with full platform stack
docker compose -f compose.container.yaml up -d
```

## Monitoring

| Endpoint                   | Description                      |
| :------------------------- | :------------------------------- |
| `GET /actuator/health`     | Liveness + readiness probes      |
| `GET /actuator/info`       | Build info and platform metadata |
| `GET /actuator/metrics`    | Application metrics              |
| `GET /actuator/prometheus` | Prometheus scrape endpoint       |
| `GET /swagger-ui.html`     | API documentation                |

## Project Structure

```
src/main/java/com/iqkv/docintelligence/salesroomservice/
├── ping/            # Reference feature slice — probes for the three security tiers
├── infrastructure/  # Cross-cutting technical concerns: Spring config, security, messaging, persistence
│   ├── config/      # @Configuration, properties, global exception handler
│   ├── security/    # Correlation-id and tenant-extraction filters, JWT claim names
│   ├── persistence/ # MyBatis / datasource configuration
│   └── messaging/   # RabbitMQ configuration and tenant provisioning consumer
├── shared/          # Cross-slice kernel — common exceptions and utilities
└── SalesRoomServiceApplication.java
```

New domain features go in as sibling packages next to `ping/` — one package per feature slice, holding its REST resources, service contract and implementation, DTOs and slice exceptions.

## License

This project is licensed under the Apache License. See the [LICENSE](LICENSE) file for details.

## Contributing

Please read our [Contributing Guidelines](.github/CONTRIBUTING.md) and [Code of Conduct](.github/CODE_OF_CONDUCT.md).

---

## 🧩 Architecture

- **Venue library** — owned by `docintelligence-catalog-service` and read straight from the shared database (same PostgreSQL database in production) via the `com.iqkv:docintelligence-venue` data layer; this service stores only salesroom-scoped records
- **API tiers** — public (`/ping`), tenant (JWT + resolved tenant context), and platform admin (`PLATFORM_ADMIN`); mirrored per feature slice
- **Security** — Spring Security + OAuth2 Resource Server; RS256 JWT validated through the IAM JWKS endpoint (deployed) or a classpath PEM key (local dev / tests); `@PreAuthorize` on protected endpoints
- **Multi-tenancy** — `ROLLOUT_MODE` (`MULTI_TENANT` | `SINGLE_TENANT`) — must be identical across all platform services; tenant data lives in a `t_{tenantKey}` schema
- **Persistence** — MyBatis with XML mappers + PostgreSQL; Liquibase manages system and per-tenant schema migrations
- **Object storage** — MinIO S3-compatible client (`OBJECTSTORAGE_*`); the `salesroom-assets` bucket stays private and clients receive presigned URLs
- **Messaging** — RabbitMQ consumer for IAM tenant provisioning events (`tenant.created` / `tenant.updated`) bound to the shared `iqkv.events` exchange; `iqkv.messaging.rabbitmq.enabled` toggle — disabled in base profile, enabled per environment
- **AI** — Ollama connectivity configured via `spring.ai.ollama.*` (`SPRING_AI_OLLAMA_BASE_URL`, `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL`, `SPRING_AI_OLLAMA_EMBEDDING_MODEL`); no AI feature slice ships yet
- **Observability** — Micrometer + Prometheus; structured JSON logging with Logstash encoder; health probes for Kubernetes readiness/liveness
- **Quality Tools** — Checkstyle, JaCoCo (60% gate), ArchUnit, commit convention enforcement

> See [AGENTS.md](AGENTS.md) for repository structure, feature-slice conventions, and agent guidelines.
