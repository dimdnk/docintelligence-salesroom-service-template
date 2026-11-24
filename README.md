# Document Intelligence Digital Sales Room 🚀

GitHub template for the iQ Key Value platform. A multi-tenant, JWT-secured Spring Boot starter for the **Digital Sales Room for Events** — the client-facing pitch your buyers can actually use.

## About

**Digital Sales Room for Events** — select venues from your library, generate a branded interactive salesroom page in one click, and send the link. Your client browses, asks questions, and approves — no PDF, no account, no back-and-forth.

- **Brief to pitch in minutes** — pick venues, click Generate, send a link. Done before the client follows up.
- **No client login required** — the client opens a link on any device, browses photos and floor plans, and approves.
- **Questions in context** — the client comments on a specific venue; you answer right there, not scattered across email threads.
- **Immutable approval record** — on approval, every field, file, and decision is locked with a timestamp. No disputes later.
- **Who it's for** — event agencies pitching venue shortlists to their clients.

This template ships the platform plumbing. The domain surfaces above are still to be built — see [Planned surfaces](#planned-surfaces).

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

## Targets

We are not building a CRM. This is a focused sales tool for one planner selling to one client, a private individual (B2C) or a small business (B2 small B). The flow has to be simple enough that neither side needs training, and data intelligence works as an assistant to both sides rather than a dashboard for a sales manager.

The test for this service is not feature parity with the established digital sales room tools. It is whether an event planner can do, with venues and their own client, the same job those tools let a generic B2B seller do: share a living page, see engagement, take questions in context, present options and prices, and end in an approval both sides trust.

The table below is the honest checklist. Left column is what buyers already expect from a sales room tool, right column is what "useful" means for us.

| Business case                        | What today's sales room tools do                                                 | What we target                                                                                                   |
| :----------------------------------- | :------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------- |
| Share a live page, not a PDF         | One branded link, updated in place, no buyer account                             | The shortlist itself as the page: venues, photos, floor plans, capacity, availability                            |
| Know when the buyer engaged          | View counts, time on page, per-recipient tracking, alerts for the seller         | Planner sees whether and when the client opened the pitch and which venues held attention                        |
| Ask and answer in context            | Buyer comments on a specific block; the thread stays attached to it              | Client comments on a venue; the planner answers in the same thread, so Q&A never drifts into email               |
| Present options side by side         | Multiple packages or tiers with pricing, selectable by the buyer                 | Venue options per venue with the planner's own package pricing and inclusions                                    |
| Let the buyer choose, not just read  | Buyer picks a package, declines alternatives, states preferences                 | Client marks each venue favourite, maybe or rejected, and the planner sees the shape of the decision             |
| Agree next steps                     | Mutual action plan: owners, dates, visible to both sides                         | Agreed steps toward the event, attached to the shortlist rather than a separate tool                             |
| Collect formal sign-off              | Approval or e-signature that closes the deal and locks the content               | Client approval of the chosen option, frozen with a timestamp, a content hash and the venue data it was based on |
| Keep everything in one place         | Content library, collateral, and the page never fragment across email and drives | Media, notes and pricing stay on the board; the client never receives a stale attachment                         |
| Separate internal from client-facing | Seller-only notes and folders hidden from the buyer                              | Planner annotations and reasoning stay private; only published content reaches the client                        |
| Control access                       | Link permissions, expiry, revocation                                             | Revocable share links with no client login, retained through the event date plus 30 days                         |

What those tools cannot give us is the input. Their pages start from a document the seller uploads. Ours starts from the planner's own venue library, so options carry the planner's real venue intelligence instead of a file somebody retyped.

### Priority 1 — four features on top of the checklist

These four decide whether the board is a page or a process. Everything above is already in the design.

| Feature              | What it means for us                                                                                                                                                            |
| :------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Plan of record       | Every step toward the event as a task with an owner (planner or client) and a date, ticked off inside the board. Replaces "agreed steps" as a loose note.                        |
| People, not one client | The client side is often several people, so each participant carries a role and their own view, with an optional email gate so the planner learns who opened the board and who a forwarded link reached. |
| Tell the planner     | Engagement only matters if it arrives. Email notifications and a digest carry opens, comments and decisions to the planner; a Slack or Teams push is a later option.              |
| Collect the details  | One form on the board for what the event needs (headcount, dietary, AV, billing), with the answers landing next to the approval instead of in email.                            |

### Priority 2 — behaviour and flow analysis

Once activity is recorded per person and per venue, the intelligence layer turns it into two readings that help both sides, not only the seller.

| Analysis   | What it answers                                                                                                                                                  |
| :--------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Behaviour  | Who looked at what, how often and how recently, per venue and per person. "The decision maker has not opened the board in nine days." "The client keeps returning to one venue and never opens the others." |
| Flow       | Where the sale stalls inside the board: shared but never opened, opened but no questions, questions answered but no selection, selection but no approval. Each stage names the next action for the planner and shows the client what still waits on them. |

> Deliberately not in scope for this service: CRM pipeline sync, prospecting, e-signature with legal weight, seller-to-seller chat, and live video calls. We are the client-facing layer of a focused sale, not a revenue platform.

## MVP

The sales room is two surfaces in two projects. The **tenant app** is where the deal owner works; the **pitch board microsite** is the short-link page the client opens, with its own UI and no login.

### Tenant app — creating a deal

1. **Name + client contact.** Deal name and the client's name and email. A second contact stays optional until more than one person is involved.
2. **Template via multi-step builder.** System templates decide the default label and value set, the form defaults, and the branding defaults. Templates are code-defined placeholders, not database rows. Templates start with: basic, wedding, corporate training, and more added later.
3. **Pick one or more venues from the catalog.** The planner searches their venue library and annotates each pick (notes, price breakdown, media). Venue data is read from the catalog read-only; the board takes a snapshot at share time so later catalog drift does not rewrite the client's page.
4. **Define form elements for collecting inputs.** Headcount, date, dietary, AV, billing. Each element has a label, a type (text, number, date, choice), a required or optional flag, and who fills it (client or planner). Answers land on the deal, not in email.
5. **Notes.** Internal notes (planner only) and shareable notes (client sees), both per deal and per venue.
6. **Send.** A short link is generated and emailed to the client contact. The pitch board goes live at the share URL.

### Pitch board microsite — what the client sees

- Venues with photos, capacity, notes, price breakdown and media.
- Per-venue reactions: favourite, maybe, rejected.
- Two-sided comments, per venue and per board.
- The form the planner asked the client to fill.
- Shareable notes.
- Approval: the client approves the selection, freezing a timestamped snapshot with a content hash and the venue data it was based on.

### What the planner sees back

- Engagement: opens, per-venue views, comments, reactions, form answers.
- Tell the planner: email notification and digest. Engagement is only useful if it arrives.
- Plan of record: next steps with an owner and a date.
- People: who opened the board, via an optional email gate.

### Build order

1. **Tenant builder first.** Freeze the data structures, the management APIs and the planner UI. Nothing is shared with the client until the planner can create, edit and review a complete deal.
2. **Pitch board microsite second.** Once the data structures are stable, build the public short-link page that reads them.

### Not in MVP

- CRM sync, prospecting, legal e-signature, seller-to-seller chat, live video calls.
- AI deal coaching, document generation, per-venue scoring. These layer on top once activity is recorded.
- White-label, SSO as a salesroom feature (SSO already lives in IAM).
- Post-approval event-execution hub. The board can continue as that hub later, but the MVP ends at approval.

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

## Tenant Schema (planned)

The service shares the docintelligence PostgreSQL instance but owns its own tables, created in the `t_{tenantKey}` schema by this service's tenant changelog when IAM emits `tenant.created`. Catalog venues are referenced by id and read read-only — no database foreign key crosses the service boundary, so pitch data stays intact even if a venue leaves the library. Assets live in the private `salesroom-assets` bucket; only their metadata is stored here.

| Table                    | Side    | Purpose                                                                                                                               |
| :----------------------- | :------ | :------------------------------------------------------------------------------------------------------------------------------------ |
| `salesroom_boards`       | planner | The salesroom itself — slug, share token, title, event context, branding, status (`DRAFT` → `SHARED` → `APPROVED` → `ARCHIVED`)       |
| `salesroom_board_venues` | planner | The picked catalog venue rows (venue id + position) with the planner's annotation and a frozen venue metadata snapshot                |
| `salesroom_options`      | planner | Planner-offered options per venue entry — package name, inclusions and price breakdown                                                |
| `salesroom_price_items`  | planner | Line items behind an option's price: label, quantity, unit price, currency                                                            |
| `salesroom_media`        | planner | Photos, floor plans and documents attached to a board, venue entry or option; the file sits in S3, the row holds the key and metadata |
| `salesroom_comments`     | both    | Two-sided Q&A — planner and client comments on a board, venue entry or option, answered in context                                    |
| `salesroom_markers`      | client  | Client's per-venue or per-option reaction — favourite, maybe, rejected                                                                |
| `salesroom_activity`     | both    | Activity feed — views, shares, comments, marker changes and approvals                                                                 |
| `salesroom_approvals`    | client  | Immutable, timestamped approval snapshot: frozen content hash plus approver identity                                                  |

> Initial, indicative list only. Tables and columns are expected to change as the slices in [Planned surfaces](#planned-surfaces) are built.

## Design Decisions (MVP)

Two rules shape every table above.

**1. References to source assets are application-level, not database constraints.**

Venue ids, asset keys and author ids are stored as plain values. No database foreign key links a sales room row to a catalog, storage or IAM record — the application validates the reference, and catalog venues are read read-only. Keeping the boundary in the database itself means the storage layer can be extracted into its own database later without rewriting constraints, and a venue, file or user disappearing from its source never cascades into pitch history.

**2. Sales room content is label/value data, not fixed columns.**

A pitch, a proposal, or any panel inside them stores its content as label + value pairs, each marked optional or required. The board renders those values through placeholders and variables defined in code, the way a template processor works, so templates are not database rows and no table grows with layouts. A template is a starting point, not a limit: the planner stays free to build their own presentation of that document or screen. Because the shape is data rather than columns:

- One or more venues can be attached to a pitch. Where the catalog values fit, they sync straight into the deal documents.
- Where they do not fit, the planner or deal owner edits any value on the record by hand.
- Values that belong to the client side, such as comments and value notes, stay editable by the planner's client as well.

The required/optional marking is what keeps that freedom usable: required values must be filled before a pitch is shared or approved.

What the two rules buy us:

- The catalog keeps its own concerns (ingestion, extraction, search, master venue). A sales room never inherits them, so a change on either side stays inside its own service.
- A different event case, or a lower level of data quality, is still just a different label and value set.
- The same board renders with different layouts, skins and themes without touching the stored data.
- What varies per agency and per deal is content and presentation. Tenancy, provisioning, storage and metrics stay fixed underneath, which is what keeps the platform operable as SaaS.

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
git clone https://github.com/IQKV/docintelligence-salesroom-service.git my-service
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
| `MESSAGING_ENABLED`                   | `false`                  | Maps to `iqkv.messaging.rabbitmq.enabled`                 |
| `SPRING_AI_OLLAMA_BASE_URL`           | `http://localhost:11434` | Ollama server URL                                         |
| `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL` | `llama3.2`               | Model name (must be pulled in Ollama)                     |
| `ROLLOUT_MODE`                        | `MULTI_TENANT`           | Platform rollout mode                                     |

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
docker build -t iqkv/docintelligence-salesroom-service:latest .

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
├── src/main/java/com/iqkv/docintelligence/salesroomservice/
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

- **Venue library** — owned by `docintelligence-catalog-service` and read straight from the shared database (same PostgreSQL database in production) via the `com.iqkv:docintelligence-venue` data layer; this service stores only salesroom-scoped records
- **API tiers** — public (`/ping`), tenant (JWT + resolved tenant context), and platform admin (`PLATFORM_ADMIN`); mirrored per feature slice
- **Security** — Spring Security + OAuth2 Resource Server; RS256 JWT validated through the IAM JWKS endpoint (deployed) or a classpath PEM key (local dev / tests); `@PreAuthorize` on protected endpoints
- **Multi-tenancy** — `ROLLOUT_MODE` (`MULTI_TENANT` | `SINGLE_TENANT`) — must be identical across all platform services; tenant data lives in a `t_{tenantKey}` schema
- **Persistence** — MyBatis with XML mappers + PostgreSQL; Liquibase manages system and per-tenant schema migrations
- **Object storage** — MinIO S3-compatible client (`OBJECTSTORAGE_*`); the `salesroom-assets` bucket stays private and clients receive presigned URLs
- **Messaging** — RabbitMQ consumer for IAM tenant provisioning events (`tenant.created` / `tenant.updated`) bound to the shared `iqkv.events` exchange; `iqkv.messaging.rabbitmq.enabled` toggle — disabled in base, enabled per profile
- **AI** — Ollama connectivity configured via `spring.ai.ollama.*` (`SPRING_AI_OLLAMA_BASE_URL`, `SPRING_AI_OLLAMA_CHAT_OPTIONS_MODEL`, `SPRING_AI_OLLAMA_EMBEDDING_MODEL`); no AI feature slice ships yet
- **Observability** — Micrometer + Prometheus; structured JSON logging with Logstash encoder; health probes for Kubernetes
- **Credential convention** — DB users follow `svc_{service}_dba`, RabbitMQ users follow `svc_{service}_rmq`
- **GitHub Integration** — Issue templates, labels, Dependabot, and CI workflows
- **Quality Tools** — Checkstyle, JaCoCo (60% gate), ArchUnit, commit convention enforcement

> See [AGENTS.md](AGENTS.md) for repository structure, feature-slice conventions, and agent guidelines.
