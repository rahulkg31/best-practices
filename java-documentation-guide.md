# Documentation Guide for Java Projects 

This expands the original guide with a small worked example for each file, so the structure is concrete rather than abstract.

| File                    | Job                              | Audience                                 | Changes when...                                 |
| ------------------------ | --------------------------------- | ------------------------------------------ | -------------------------------------------------- |
| `README.md`             | Get someone running in 5 minutes | New devs, contributors                   | Stack, build commands, project structure change |
| `docs/CONFIGURATION.md` | Every setting, exhaustively      | Ops/DevOps, whoever deploys it           | Any property is added/removed/changed           |
| `docs/API.md`           | Every endpoint, exhaustively     | API consumers, frontend/integration devs | Any endpoint is added/changed                   |

---

## 1. README.md

### Required sections, in order
1. One-line description
2. Tech stack (with versions)
3. Project layout (tree, 1-line comments)
4. Quick Config (optional, 5–8 rows)
5. Running locally
6. Running with Docker
7. Building the distribution
8. Running tests
9. Links out

### Example

```markdown
# Order Service

Handles order creation, status tracking, and cancellation for the storefront.

## Tech Stack
- Java 17
- Spring Boot 3.2
- PostgreSQL 15
- Gradle 8.5

## Project Layout
order-service/
├── src/main/java/com/acme/orders/   # domain, controllers, services
├── src/main/resources/              # application.yml, db migrations
├── src/test/java/                   # unit + integration tests
└── docker/                          # local docker-compose setup

## Quick Config

| Property                     | Env var         | Default     |
|-------------------------------|-----------------|-------------|
| server.port                   | SERVER_PORT     | 8080        |
| spring.datasource.url         | DB_URL          | (required)  |
| spring.datasource.password    | DB_PASSWORD     | (required)  |
| spring.profiles.active        | SPRING_PROFILES_ACTIVE | dev  |

See [CONFIGURATION.md](docs/CONFIGURATION.md) for the full list.

## Running Locally
\`\`\`bash
./gradlew bootRun
\`\`\`
API available at `http://localhost:8080`.

## Running with Docker
\`\`\`bash
docker compose up
\`\`\`

## Building the Distribution
\`\`\`bash
./gradlew bootJar
\`\`\`
Produces `build/libs/order-service.jar`.

## Running Tests
\`\`\`bash
./gradlew test        # unit tests
./gradlew integrationTest  # integration tests, requires Docker
\`\`\`

## Docs
- [CONFIGURATION.md](docs/CONFIGURATION.md) — all settings
- [API.md](docs/API.md) — endpoint reference
```

---

## 2. CONFIGURATION.md

### Required sections
1. How configuration is layered
2. One table per logical group
3. Standard columns: `Property | Env var override | Default | Description`
4. "Where to add a new property" checklist

### Example

```markdown
# Configuration Reference

## Layering
Spring Boot resolves properties in this order (later wins):
`application.yml` → `application-{profile}.yml` → external config file (`--spring.config.location`) → environment variables → command-line args.

## Custom Properties (`com.acme.orders.OrderProperties`)

| Property                  | Env var                | Default | Description |
|-----------------------------|---------------------------|---------|--------------|
| orders.cancellation-window | ORDERS_CANCELLATION_WINDOW | 15m    | How long after placement an order can still be cancelled by the customer. Validated as an ISO-8601 duration; invalid values fail startup. |
| orders.max-items-per-order | ORDERS_MAX_ITEMS_PER_ORDER | 50     | Hard cap on line items per order, enforced at the API layer (400 if exceeded). |

Bound via `@ConfigurationProperties(prefix = "orders")` — see `OrderProperties.java` for the full class if a setting isn't covered here.

## Database

| Property                     | Env var             | Default (dev) | Default (prod) | Description |
|--------------------------------|------------------------|----------------|------------------|--------------|
| spring.datasource.url          | DB_URL                 | jdbc:postgresql://localhost:5432/orders | (none — required) | JDBC connection string. |
| spring.datasource.password     | DB_PASSWORD            | postgres       | (none — required) | **Always override this — never run with the default in production.** |
| spring.datasource.hikari.maximum-pool-size | DB_MAX_POOL_SIZE | 10 | 20 | Max concurrent DB connections this instance holds open — tune against your DB's connection limit divided by instance count. |

## Actuator

| Property                                    | Env var                     | Default        | Description |
|-----------------------------------------------|--------------------------------|-----------------|--------------|
| management.endpoints.web.exposure.include    | MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE | health,info | Expands to `*` in some dev setups — confirm this is scoped down before deploying, since `*` exposes `/actuator/env` and similar sensitive endpoints. |

## Where to Add a New Property
1. Add the field to the relevant `@ConfigurationProperties` class (or confirm it's a native Spring property).
2. Add a default to `application.yml` (and `application-prod.yml` if it differs).
3. Add a row to the appropriate table above, including the env var form.
4. If security-sensitive, add the "always override" warning.
```

---

## 3. API.md

### Decide first
Generate the OpenAPI spec (springdoc-openapi); hand-write a short `API.md` for conventions + a link to Swagger UI, not a full duplicate endpoint list.

### Required sections
1. Base URL per environment
2. Auth
3. Conventions (pagination, errors, versioning)
4. Endpoint reference (table linking out, or full inline detail for small APIs)
5. Example `curl` requests

### Example

```markdown
# API Reference

Full interactive spec: `/swagger-ui.html` (see CONFIGURATION.md for `springdoc.*` settings).

## Base URLs
| Environment | URL |
|-------------|-----|
| Local       | http://localhost:8080/api/v1 |
| Staging     | https://staging.acme.com/api/v1 |
| Production  | https://api.acme.com/api/v1 |

## Auth
Bearer JWT in the `Authorization` header. Tokens are issued by the auth service and expire after 1 hour.

## Conventions

**Pagination**: `?page=0&size=20`, response wrapped as:
\`\`\`json
{ "content": [...], "page": 0, "size": 20, "totalElements": 137 }
\`\`\`

**Errors**: 4xx/5xx return:
\`\`\`json
{ "status": 404, "error": "Not Found", "message": "Order 123 not found", "timestamp": "..." }
\`\`\`

Validation errors (400, from `@RestControllerAdvice` around `MethodArgumentNotValidException`):
\`\`\`json
{ "status": 400, "error": "Validation Failed", "fields": { "quantity": "must be >= 1" } }
\`\`\`

**Versioning**: path-based (`/api/v1/...`). Old versions supported for 6 months after a new version ships.

## Endpoints

### Orders
| Method | Path | Purpose |
|--------|------|---------|
| POST   | /orders | Create an order |
| GET    | /orders/{id} | Get order by ID |
| DELETE | /orders/{id} | Cancel an order (within cancellation window) |

Full request/response schemas: Swagger UI → "Orders" tag.

### Example: Create an order
\`\`\`bash
curl -X POST https://api.acme.com/api/v1/orders \\
  -H "Authorization: Bearer $TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"items": [{"sku": "SKU-1", "quantity": 2}]}'
\`\`\`

Success (201):
\`\`\`json
{ "id": "ord_123", "status": "PENDING", "items": [{"sku": "SKU-1", "quantity": 2}] }
\`\`\`

Order-specific errors: `409 Conflict` if the same idempotency key is reused with a different payload.
```

