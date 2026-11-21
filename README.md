# Document Intelligence Pitch Service 🚀

GitHub template for the iQ Key Value platform. A multi-tenant, JWT-secured Spring Boot starter for the **Digital Sales Room for Events** — the client-facing pitch your buyers can actually use.

## About

**Digital Sales Room for Events** — select venues from your library, generate a branded interactive pitch page in one click, and send the link. Your client browses, asks questions, and approves — no PDF, no account, no back-and-forth.

- **Brief to pitch in minutes** — pick venues, click Generate, send a link. Done before the client follows up.
- **No client login required** — the client opens a link on any device, browses photos and floor plans, and approves.
- **Questions in context** — the client comments on a specific venue; you answer right there, not scattered across email threads.
- **Immutable approval record** — on approval, every field, file, and decision is locked with a timestamp. No disputes later.
- **Who it's for** — event agencies pitching venue shortlists to their clients.

This template ships the platform plumbing. The domain surfaces above are still to be built — see [Planned surfaces](#planned-surfaces).

### What this starter includes

- **Venue library consumption** — venues come from `docintelligence-catalog-service` (read-only); this service owns pitches, not the venue library
- **Feature-sliced layout** — one package per vertical feature slice; `ping` ships as the reference slice
- **Three security tiers** — public, tenant JWT (`MEMBER` / `TENANT_OWNER` / `ADMIN`), and platform admin (`PLATFORM_ADMIN`)
- **Multi-tenant isolation** — tenant context resolved from the `X-Tenant-ID` header or the JWT `tenant_id` claim before every request
- **Persistence** — PostgreSQL via MyBatis (XML mappers); Liquibase manages system and per-tenant schema migrations
- **Object storage** — MinIO (S3-compatible) client bound to a private `pitch-assets` bucket; clients receive presigned URLs
- **Messaging** — RabbitMQ consumer for IAM tenant provisioning events (`tenant.created` / `tenant.updated`)
- **AI connectivity** — Spring AI Ollama starter pre-wired via `spring.ai.ollama.*` (base URL and models configurable per environment)
- **Observability** — Micrometer + Prometheus + structured JSON logging (Logstash encoder)
- **Quality tooling** — Checkstyle, JaCoCo, ArchUnit, Husky git hooks, commitlint

## Template Usage

1. Click **[Use this template](https://github.com/IQKV/docintelligence-pitch-service/generate)** to create your repository
2. Replace the remaining `pitchservice` / `PitchService` tokens with your service name (see the token map in `foundation-microservice-project-layout`)
3. Update `pom.xml` — `artifactId`, `name`, `description`, `start-class`
4. Rename the Java package from `com.iqkv.docintelligence.pitchservice` to your package
5. Rename `docker/postgres/init-pitch.sql` and update the `pitchservice` database and `svc_pitch_*` credentials to match
6. Rename the tenant-provisioning queue and DLQ in `RabbitMQConfig` (`docintelligence.pitch.tenant.provisioning`) to your service name
7. Remove unused dependencies from `pom.xml` (e.g. `spring-ai-starter-model-ollama` if no LLM integration is needed)
8. Add your feature slices as sibling packages under `com.iqkv.docintelligence.pitchservice`

## Quick Links

- [API Documentation](./docs/api/README.md)
- [Architecture Overview](./docs/architecture/README.md)
- [Deployment Guide](./docs/deployment/README.md)
- [Contributing Guidelines](.github/CONTRIBUTING.md)

## API

Base path: `/api/v1/pitch`

### Ping

| Method | Path           | Auth                 | Description                                                           |
| :----- | :------------- | :------------------- | :-------------------------------------------------------------------- |
| `GET`  | `/ping`        | public               | Unauthenticated reachability probe — no JWT or tenant header required |
| `GET`  | `/tenant/ping` | JWT + `X-Tenant-ID`  | Tenant-scoped probe — echoes the resolved tenant key                  |
| `GET`  | `/admin/ping`  | JWT `PLATFORM_ADMIN` | Admin probe — echoes the caller's `user_id` claim                     |

### Planned surfaces

`ping` is the only shipped surface — no domain endpoints exist yet. The slices below are the ones the Digital Sales Room brief calls for:

- **Venue selection** — search the venue library and add venues to a pitch
- **Pitch generation** — one click renders the selected venues as a branded, interactive client page
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

## Prerequisites

- JDK 25 (Eclipse Temurin)
- Maven 3.9+
- Node.js >= 22.15.0 & pnpm >= 10.33.2 (git hooks)
- Docker & Docker Compose
- [Ollama](https://ollama.com) running locally with at least one model pulled (e.g. `ollama pull llama3.2`)

## Quick Start

```bash
# Clone or use as template
git clone https://github.com/IQKV/docintelligence-pitch-service.git my-service
cd my-service

# Install git hooks
pnpm install

# Pull a model into Ollama (one-time)
ollama pull llama3.2

# Copy environment variables
cp .env.example .env.local
# Defaults point to Ollama on localhost:11434 — no API key needed for local dev

# Start dependencies (PostgreSQL, RabbitMQ, MinIO, etc.)
docker compose up -d

# Run the service
./mvnw spring-boot:run -Pdev
# → API:          http://localhost:8080
# → Actuator:     http://localhost:8081/actuator/health
# → Swagger:      http://localhost:8080/swagger-ui.html
# → MinIO:        http://localhost:9003 (console)
# → DbGate:       http://localhost:3100
```

## Environment Variables

| Variable                              | Default                  | Description                               |
| :------------------------------------ | :----------------------- | :---------------------------------------- |
| `DB_HOST`                             | `localhost`              | PostgreSQL host                           |
| `DB_PORT`                             | `5432`                   | PostgreSQL port                           |
| `DB_NAME`                             | `pitchservice`           | Database name                             |
| `DB_USERNAME`                         | `svc_pitch_dba`          | Database user                             |
| `DB_PASSWORD`                         | `svc_pitch_dba`          | Database password                         |
| `RABBITMQ_HOST`                       | `localhost`              | RabbitMQ host                             |
| `RABBITMQ_PORT`                       | `5672`                   | RabbitMQ AMQP port                        |
| `RABBITMQ_USERNAME`                   | `svc_pitch_rmq`          | RabbitMQ user                             |
| `RABBITMQ_PASSWORD`                   | `svc_pitch_rmq`          | RabbitMQ password                         |
| `OBJECTSTORAGE_ENDPOINT`              | `http://localhost:9000`  | S3-compatible endpoint (MinIO in SIT/UAT) |
| `OBJECTSTORAGE_PUBLIC_ENDPOINT`       | _(empty)_                | Public URL used to rewrite presigned URLs |
| `OBJECTSTORAGE_ACCESS_KEY`            | `iqkv`                   | S3 access key                             |
| `OBJECTSTORAGE_SECRET_KEY`            | `iqkv_password`          | S3 secret key                             |
| `OBJECTSTORAGE_BUCKET_NAME`           | `pitch-assets`           | Private bucket holding pitch assets       |
| `OBJECTSTORAGE_REGION`                | `us-east-1`              | S3 region                                 |
| `OBJECTSTORAGE_SSL`                   | `false`                  | Use HTTPS for the S3 endpoint             |
| `MESSAGING_ENABLED`                   | `false`                  | Maps to `iqkv.messaging.rabbitmq.enabled` |
| `SPRING_AI_OLLAMA_BASE_URL`           | `http://localhost:11434` | Ollama server URL                         |
| `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL` | `llama3.2`               | Model name (must be pulled in Ollama)     |
| `ROLLOUT_MODE`                        | `MULTI_TENANT`           | Platform rollout mode                     |

> Copy `.env.example` to `.env.local` / `.env.uat` / `.env.prd` and fill in values per environment.
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
docker build -t iqkv/docintelligence-pitch-service:latest .

# Run full stack (service + dependencies)
docker compose -f compose.container.yaml up -d
```

## Monitoring

| Endpoint                   | Description                 |
| :------------------------- | :-------------------------- |
| `GET /actuator/health`     | Liveness + readiness probes |
| `GET /actuator/metrics`    | Application metrics         |
| `GET /actuator/prometheus` | Prometheus scrape endpoint  |
| `GET /swagger-ui.html`     | API documentation           |
| `GET /` (on port 9003)     | MinIO Console               |
| `GET /` (on port 3000)     | Grafana dashboards          |
| `GET /` (on port 9090)     | Prometheus UI               |

## Project Structure

```
.
├── docker/                           # Local stack config (postgres, prometheus, grafana)
├── docs/                             # API, architecture, deployment
├── src/main/java/com/iqkv/docintelligence/pitchservice/
│   ├── ping/                         # Reference feature slice — probes for the three security tiers
│   ├── infrastructure/               # Cross-cutting technical concerns, no business logic
│   │   ├── config/                   # Spring config, properties, global exception handler
│   │   ├── security/                 # Correlation-id and tenant-extraction filters, JWT claim names
│   │   ├── persistence/              # MyBatis / datasource configuration
│   │   └── messaging/                # RabbitMQ configuration and tenant provisioning consumer
│   └── shared/                       # Cross-slice kernel — common exceptions and utilities
└── src/main/resources/
    ├── application.yml               # Base configuration
    ├── application-{local,sit,uat,prd}.yml
    ├── db/changelog/{system,tenant}/ # Liquibase changelogs
    └── i18n/                         # Validation and message bundles
```

New domain features go in as sibling packages next to `ping/` — one package per feature slice, holding its REST resources, service contract and implementation, DTOs and slice exceptions.

## License

This project is licensed under the Apache License. See the [LICENSE](LICENSE) file for details.

## Branch Strategy

> **Template note:** The strategy below is the default shipped with this template. Replace it with whatever model fits your team — update branch names, CI triggers, and protection rules to match.

This template uses **scaled trunk-based development**:

- `dev` is the trunk — always stable and deployable
- Short-lived feature/bugfix branches merge into `dev` via pull request
- Releases are cut from `dev` as annotated tags (`v1.2.0`) — no separate release branch

```
dev  ← trunk
├── feature/*
├── bugfix/*
├── improvement/*
├── hotfix/*
└── rfc/*
```

## Contributing

Please read our [Contributing Guidelines](.github/CONTRIBUTING.md) and [Code of Conduct](.github/CODE_OF_CONDUCT.md).

---

## 🧩 Architecture

- **Venue library** — owned by `docintelligence-catalog-service`; this service reads venue data and stores only pitch-scoped records
- **API tiers** — public (`/ping`), tenant (JWT + resolved tenant context), and platform admin (`PLATFORM_ADMIN`); mirrored per feature slice
- **Security** — Spring Security + OAuth2 Resource Server; RS256 JWT validated through the IAM JWKS endpoint (deployed) or a classpath PEM key (local dev / tests); `@PreAuthorize` on protected endpoints
- **Multi-tenancy** — `ROLLOUT_MODE` (`MULTI_TENANT` | `SINGLE_TENANT`) — must be identical across all platform services; tenant data lives in a `t_{tenantKey}` schema
- **Persistence** — MyBatis with XML mappers + PostgreSQL; Liquibase manages system and per-tenant schema migrations
- **Object storage** — MinIO S3-compatible client (`OBJECTSTORAGE_*`); the `pitch-assets` bucket stays private and clients receive presigned URLs
- **Messaging** — RabbitMQ consumer for IAM tenant provisioning events (`tenant.created` / `tenant.updated`) bound to the shared `iqkv.events` exchange; `iqkv.messaging.rabbitmq.enabled` toggle — disabled in base, enabled per profile
- **AI** — Ollama connectivity configured via `spring.ai.ollama.*` (`SPRING_AI_OLLAMA_BASE_URL`, `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL`, `SPRING_AI_OLLAMA_EMBEDDING_MODEL`); no AI feature slice ships yet
- **Observability** — Micrometer + Prometheus; structured JSON logging with Logstash encoder; health probes for Kubernetes
- **Credential convention** — DB users follow `svc_{service}_dba`, RabbitMQ users follow `svc_{service}_rmq`
- **GitHub Integration** — Issue templates, labels, Dependabot, and CI workflows
- **Quality Tools** — Checkstyle, JaCoCo (60% gate), ArchUnit, commit convention enforcement

> See [AGENTS.md](AGENTS.md) for repository structure, feature-slice conventions, and agent guidelines.
