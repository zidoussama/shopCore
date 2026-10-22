# ShopCore

ShopCore is a learning-oriented commerce platform with independently owned
Spring Boot services and a React client. This phase establishes a buildable
foundation: structure, application startup, health/OpenAPI, infrastructure,
documentation, and CI. Business features are awaiting review and approval.

## Technology choices

| Tool | Selection | Reason |
| --- | --- | --- |
| Java | 17 LTS | Already installed; enforced and supported by Boot 3.5 |
| Maven | 3.9.x; verified with 3.9.16 | One backend reactor, no shared business library |
| Spring Boot | 3.5.16 | Stable 3.5 patch with JUnit 5; all services inherit one BOM |
| Spring Cloud | 2025.0.3 | Compatible Boot 3.5 release train, Gateway only |
| Springdoc OpenAPI | 2.8.17 | Compatible with Spring Boot 3.5 |
| Flyway / JUnit | 11.7.2 / 5.12.2 | Managed by the Boot BOM |
| Node / npm | Node 24.x LTS; .node-version 24.18.0 / npm 11 | Matches the inspected environment |
| React / React DOM | 19.3.1 | Stable, exact versions |
| TypeScript / Vite | 6.0.3 / 8.3.5 | Strict TypeScript and compatible Node 24 build |
| React Vite plugin | 6.1.2 | Vite 8 compatible |
| React Router / TanStack Query | 7.18.4 / 5.104.5 | Used only for requested router/provider scaffolding |
| MySQL / Redis | 8.4 LTS / 7.4 | Local Compose infrastructure, compatible maintenance tags |
| RabbitMQ | Deferred | No asynchronous workflows exist yet |

Versions were checked against Maven Central and npm metadata during setup.
The frontend lockfile pins transitive dependencies. Backend dependency versions
come from the Boot/Cloud BOMs; no snapshot dependencies are selected.
Compose image tags receive compatible patches; pin tested image digests before release.

## Prerequisites

- JDK 17 on JAVA_HOME and PATH (Maven verifies the Java major version).
- Maven 3.9.x on PATH; no Maven wrapper is added in this first scaffold.
- Node 24.x and npm 11. A Node version manager can use .node-version.
- Docker Desktop with the Linux engine running, and Docker Compose.
- Network access to Maven Central, npm, and Docker Hub for first-time downloads.

The inspected machine has all CLI tools installed, but Docker's daemon was
stopped. The directory is currently an empty folder inside an existing parent
Git repository. All additions are confined here; unrelated parent work is preserved.
Treat this directory as ShopCore's repository root when publishing it.

## Repository layout

```text
shopcore/                         (this workspace, currently named BigDataProjet)
|-- backend/
|   |-- pom.xml                   (parent and nine-module Maven reactor)
|   |-- api-gateway/
|   |-- identity-service/
|   |-- catalog-service/
|   |-- cart-service/
|   |-- inventory-service/
|   |-- order-service/
|   |-- payment-service/
|   |-- shipping-service/
|   |-- notification-service/
|-- frontend/                     (React, TypeScript, Vite)
|-- infrastructure/
|   |-- docker/
|   |-- mysql/init/
|   |-- redis/
|   |-- rabbitmq/
|   |-- observability/
|   `-- scripts/
|-- docs/
|   |-- architecture/
|   |-- adr/
|   |-- api/
|   `-- diagrams/
|-- tests/
|   |-- e2e/
|   `-- performance/
|-- .github/workflows/ci.yml
|-- compose.yaml
|-- .env.example
`-- README.md
```

All nine backend projects are under backend/. Each business service has
domain, application, infrastructure, presentation, test layers, and a Flyway
migration directory. Concise package-info.java documents reserved packages;
.gitkeep preserves empty frontend and test folders. See docs/repository-tree.txt
for the complete source tree.

## Responsibilities and local ports

| Component | Port | Future responsibility / data ownership |
| --- | --- | --- |
| Frontend dev / preview | 5173 / 4173 | Minimal branded default page |
| API Gateway | 8080 | Routing, correlation IDs, authentication filters, Redis rate limiting; no business data |
| Identity Service | 8081 | Users, credentials, roles, refresh sessions, authentication, and account profiles. |
| Catalog Service | 8082 | Products, categories, descriptions, images, and published prices. |
| Cart Service | 8083 | Active shopping carts and cart items. |
| Inventory Service | 8084 | Stock quantities, reservations, and stock movements. |
| Order Service | 8085 | Order lifecycle, history, product/price snapshots, and checkout coordination. |
| Payment Service | 8086 | Payment attempts, provider references, refunds, and webhook processing. |
| Shipping Service | 8087 | Shipments and tracking events. |
| Notification Service | 8088 | Notification templates, preferences, and delivery attempts. |
| MySQL | 3306 | Eight isolated service schemas, persistent named volume |
| Redis | 6379 | Future cache/rate-limit support, persistent named volume |

These service responsibilities are plans only. Backend ports can be overridden
with SERVER_PORT. MYSQL_PORT and REDIS_PORT configure host ports.
Compose publishes infrastructure ports on 127.0.0.1 only.

## Local setup and infrastructure

From this directory in PowerShell:
```powershell
Copy-Item .env.example .env
# Edit .env and replace every placeholder with distinct local passwords.
docker compose config --quiet
docker compose up -d
docker compose ps
```

On a POSIX shell, use cp .env.example .env for the copy, then the same Docker
commands. Never commit .env. Use at least 16 letters/digits/underscores/hyphens
for local database passwords; the initializer validates this development convention.
The supplied .env.example contains placeholders, not real credentials.

MySQL's initial volume setup creates only empty shopcore_identity,
shopcore_catalog, shopcore_cart, shopcore_inventory, shopcore_order,
shopcore_payment, shopcore_shipping, and shopcore_notification schemas, each
with a distinct restricted user. No business tables exist.
Database credentials and schema initialization are first-run operations:
editing .env does not rotate existing MySQL users or rerun init scripts.

Use docker compose logs mysql redis to inspect startup and docker compose down
to stop infrastructure while retaining named volumes. docker compose down -v
deletes local database/cache data; use it only when that data may be discarded.
Application containers are not configured. RabbitMQ is not started.

Validate Compose with placeholders without creating .env or starting containers:
```powershell
docker compose --env-file .env.example config --quiet
```

## Backend checks and startup

From the repository root:
```powershell
mvn -B -ntp -f backend/pom.xml verify
mvn -f backend/catalog-service/pom.xml spring-boot:run
```

verify builds all nine executable applications and runs nine real HTTP smoke
integration tests through Maven Failsafe. Tests use random ports and no database.
They verify UP health, empty business-service OpenAPI contracts, and absent
Gateway routes. There are no domain unit tests because no rules exist yet.
Reserved test packages are ready for future domain/application tests.

The default scaffold profile excludes datasource and Flyway auto-configuration,
so each service starts without infrastructure. Gateway runs without database
configuration in every profile. Every application exposes /actuator/health;
business services also expose /v3/api-docs and /swagger-ui.html. These are
development facilities, with no business endpoints and no authentication yet.

To run Catalog against the local MySQL instance, export values from your .env.
Spring Boot does not automatically read Compose's .env file:
```powershell
# From the repository root, load only KEY=VALUE entries from your local .env.
Get-Content .env | ForEach-Object {
    if ($_ -match '^([A-Z][A-Z0-9_]*)=(.*)$') {
        [Environment]::SetEnvironmentVariable($matches[1], $matches[2], 'Process')
    }
}
$env:SPRING_PROFILES_ACTIVE = 'local'
mvn -f backend/catalog-service/pom.xml spring-boot:run
```

The simple parser expects the plain values shown in .env.example, without
quotes. On POSIX: set -a; . ./.env; set +a, then export SPRING_PROFILES_ACTIVE=local.
Each service uses its own *_DB_PASSWORD; MYSQL_HOST defaults to localhost.
The local profile enables JDBC/Flyway, with no invented migrations. Flyway may
create its internal schema-history metadata. Authentication and database
entities are not implemented.

## Frontend checks and startup

```powershell
Set-Location frontend
npm ci
npm run build
npm run dev
```

npm run build performs strict TypeScript checks and a production Vite build.
npm run check checks TypeScript only; npm run preview serves the built page.
The default page has no backend calls, storefront, forms, or checkout.
No frontend, end-to-end, or performance tests exist yet.

## CI and validation

.github/workflows/ci.yml performs the full Maven verify, npm ci plus build,
and Compose configuration validation. It does not deploy or claim nonexistent
test suites passed. GitHub discovers workflows only in the Git repository
root .github/workflows: the workflow becomes active when this directory is
published as ShopCore's root. The existing parent workflow is preserved.

See docs/validation.md for commands actually run, results, and unavailable
checks. Integration test instructions in backend/README.md describe future
MySQL Testcontainers coverage once real migrations/adapters exist.

## Current limits and next milestone

No CRUD, authentication flows, entities, checkout logic, payment provider,
broker events, cache adapters, Gateway filters, application Dockerfiles, or
production deployment exists. Architecture/API documents describe future
contracts and responsibilities. OpenAPI currently has empty paths.

Review and approve this scaffold before feature development. The next milestone
is a Catalog vertical slice: admin creates a product, Catalog persists it in its
own MySQL schema, the Gateway exposes the read route, and React displays it.
Define that slice's contract, security scope, first Flyway migration, use cases,
adapters, and behavior tests after approval. Scaffolding stops here.
