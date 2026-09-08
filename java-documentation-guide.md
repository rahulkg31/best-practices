# Documentation Guide for Java Projects

| File                    | Job                              | Audience                                 | Changes when...                                 |
| ----------------------- | -------------------------------- | ---------------------------------------- | ----------------------------------------------- |
| `README.md`             | Get someone running in 5 minutes | New devs, contributors                   | Stack, build commands, project structure change |
| `docs/CONFIGURATION.md` | Every setting, exhaustively      | Ops/DevOps, whoever deploys it           | Any property is added/removed/changed           |
| `docs/API.md`           | Every endpoint, exhaustively     | API consumers, frontend/integration devs | Any endpoint is added/changed                   |

## 1. README.md — instructions

### Required sections, in this order

1. **One-line description** — what the project *is*, not what it's built with.
2. **Tech stack** — bullet list, versions included (`Java 17`, `Spring Boot 3.2`, not just "Java").
3. **Project layout** — a tree, 1-line comment per folder. Enough to navigate, not a full file listing.
4. **Quick Config** (optional but recommended) — 5–8 rows: only the properties someone *must* touch to get it running (DB host/password, port, active profile). Link out to the full `CONFIGURATION.md` for everything else. This is the one place a *little* config detail belongs in the README — just enough that a reader doesn't have to open a second file before their first successful run.
5. **Running locally** — the actual commands, copy-pasteable, with expected result ("API available at `http://localhost:8080`").
6. **Running with Docker** — if applicable, same treatment.
7. **Building the distribution** — how to produce the customer-facing artifact, if this project ships one.
8. **Running tests** — one command, one line on what's covered (unit vs integration).
9. **Links out** — to `CONFIGURATION.md`, `API.md`, `INSTALL.md`, `CHANGELOG.md`. This is the connective tissue between the three docs — put it near the top or bottom, not buried.

------

## 2. CONFIGURATION.md — instructions

### Required sections

1. **How configuration is layered** — explain the override order explicitly (e.g. `application.yml` → `application-{profile}.yml` → external file → env vars). This one paragraph prevents 80% of "why isn't my setting taking effect" questions.

2. **One table per logical group** — not one giant table. Group by: your own custom properties first (most relevant to readers), then framework/infra properties (database, web server, logging, health checks, etc.) grouped by concern.

3. Each table needs these columns, always in this order:

   | Property | Env var override | Default | Description |
   | -------- | ---------------- | ------- | ----------- |
   |          |                  |         |             |

   For Java/Spring Boot specifically, always show **both** the dotted property name (`spring.datasource.password`) and its environment variable form (`DB_PASSWORD` or `SPRING_DATASOURCE_PASSWORD`) — these look nothing alike and this mapping is the single most-needed lookup in the whole document.

4. **"Where to add a new property"** section at the end — a 3–4 step checklist so the doc stays current as the codebase grows. Docs without an update ritual go stale within a quarter.

### Writing good descriptions

- State what happens, not just what it is. Bad: "Connection pool size." Good: "Max concurrent DB connections this instance holds open — tune against your DB's connection limit divided by instance count."
- Call out anything security-sensitive explicitly: "**Always override this — never run with the default in production.**"
- If a property has validation (min/max, enum values), say so — it saves someone a failed-startup debugging session.

### Java/Spring Boot specifics to always include

- Whether the setting is bound via `@ConfigurationProperties` (custom, in your own code) or is a native Spring Boot property (external, someone else's docs apply) — readers need to know where to go look at source if the table isn't enough.
- Actuator endpoint exposure (`management.endpoints.web.exposure.include`) — this is a frequent security misconfiguration if undocumented.
- Profile-specific defaults — show the same property's value across `dev` vs `prod` in the same row when they differ, so nobody assumes dev behavior in production.

------

## 3. API.md — instructions

### Decide first: hand-written or generated?

- **Generated (springdoc-openapi / swagger-annotations)** is almost always better for REST APIs — it can't drift from the actual code, and you get Swagger UI for free. Use this as the primary source of truth.
- **Hand-written `API.md`** still earns its place for: a human-readable overview (auth flow, pagination convention, error format) that OpenAPI JSON doesn't communicate well on its own, and as a stable reference for consumers who don't want to run Swagger UI.

Best practice: **generate the OpenAPI spec, hand-write a short API.md that explains the conventions and links to the generated spec/Swagger UI** for the exhaustive endpoint list. Don't hand-maintain a full endpoint-by-endpoint doc alongside annotations — the two will drift within a month.

### Required sections for the hand-written API.md

1. **Base URL** per environment (local, staging, prod) if they differ.
2. **Auth** — how to authenticate, where the token/key goes, token lifetime if relevant.
3. **Conventions** — these save more support time than any individual endpoint doc:
   - Pagination shape (page/size params, response envelope)
   - Error response shape (status codes used, error body schema, one example)
   - Versioning scheme (`/api/v1/...`, header-based, etc.) and deprecation policy
4. **Endpoint reference** — either a table linking to Swagger UI per resource, or, for a small API, full inline detail per endpoint:
   - Method + path
   - One-line purpose
   - Request body / query params (name, type, required?, constraints)
   - Success response (status code + example body)
   - Error responses specific to this endpoint (404, 409, etc. beyond the generic ones)
5. **Example requests** — at least one full `curl` example per resource. This is what people actually copy-paste; don't skip it even if you have Swagger UI.

### Java/Spring Boot specifics to always include

- Validation error shape — Spring's `MethodArgumentNotValidException` produces a specific structure (usually via a `@RestControllerAdvice`); show it once so consumers know what a 400 looks like without guessing from Spring's defaults.
- Where the live OpenAPI JSON and Swagger UI are actually hosted (`/v3/api-docs`, `/swagger-ui.html` or wherever `springdoc.*` is configured — cross-reference CONFIGURATION.md rather than repeating the values).

------
