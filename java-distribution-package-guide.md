# Java Distribution Package

A generic, reusable structure for packaging a Java enterprise application as a customer/client-facing release artifact (`.tar.gz` / `.zip`), with optional Docker-based deployment.

```
{product-name}-{version}/                           # e.g. my-enterprise-app-1.4.2/
│
├── bin/                                            # Executable scripts — how the customer starts/stops the app
│   ├── start.sh                                    # Linux/Mac start script
│   ├── start.bat                                   # Windows start script
│   ├── stop.sh
│   ├── stop.bat
│   └── {product}-service.sh                        # Optional: systemd/init.d unit template
│
├── lib/                                            # The application itself
│   └── {product-name}-{version}.jar                # Fat/uber jar — all deps bundled, single java -jar invocation
│
├── config/                                         # Everything the customer is expected to edit
│   ├── application.yml                             # Non-sensitive defaults (safe as shipped)
│   ├── application-prod.yml.template               # Sensitive values — customer copies to application-prod.yml and fills in
│   ├── logback-spring.xml                          # Logging config (rotation, level, output path)
│   └── certs/                                      # Empty — customer drops SSL certs here if TLS is terminated at the app
│
├── logs/                                           # Empty at ship time — created for the customer, populated at runtime
│   └── .gitkeep
│
├── data/                                           # Empty at ship time — app's runtime data/uploads/working files
│
├── db/
│   └── migration/                                  # Flyway/Liquibase scripts — shipped for DBA review/manual apply
│       ├── V1__init.sql                            # if the customer's DB policy requires a human to run migrations
│       └── V2__....sql                             # rather than letting the app auto-migrate on startup
│
├── docker/                                         # Everything needed to run the app in a container
│   ├── Dockerfile                                  # Multi-stage build: builds/copies the jar, sets up runtime image
│   ├── docker-compose.yml                          # App + dependencies (DB, cache) for a full local/on-prem stack
│   ├── docker-compose.prod.yml                     # Optional: prod overrides (resource limits, restart policy, networks)
│   ├── .env.template                               # Customer copies to .env, fills in secrets/ports — mirrors application-prod.yml.template
│   ├── entrypoint.sh                               # Container entrypoint — env-to-config translation, wait-for-db, migration toggle
│   └── healthcheck.sh                              # Used by HEALTHCHECK in Dockerfile / compose healthcheck block
│
├── docs/                                           # Everything the customer reads (see companion doc for what goes in each)
│   ├── INSTALL.md                                  # Full install/upgrade/troubleshooting guide (bare-metal/jar path)
│   ├── CONFIGURATION.md                            # Every property: description, default, env var override
│   ├── API.md                                      # API reference/conventions (if applicable)
│   ├── CHANGELOG.md                                # Version history — what changed, what to watch for on upgrade
│   └── openapi.json (or swagger.yaml)              # Generated machine-readable API spec, if applicable
│
├── scripts/                                        # Operational helper scripts — day-2 operations, not startup
│   ├── install.sh                                  # One-time setup: creates dirs, copies config template
│   ├── healthcheck.sh                              # Verifies the app is actually up (hits /health or equivalent) — bare-metal path
│   └── backup.sh                                   # Backs up data/config (documents that DB backup is separate)
│
├── README.md                                       # Quick start: install → configure → start → verify
└── VERSION                                         # Plain text, single line: the version string, nothing else
```



## What should NEVER ship in a customer package

- Real secrets, passwords, or API keys of any kind (including test/demo ones) — this includes a filled-in `docker/.env`, not just `application-prod.yml`
- Your internal source code, `.git/` history, or build scripts (`pom.xml`, CI configs)
- Developer-only tooling (test data generators, load-test scripts, internal debug endpoints left enabled)
- Populated `logs/`/`data/` directories from your own testing, or leftover Docker volumes/images from your build machine
- A README written for contributors instead of one written for the person installing this



## Naming and versioning conventions

| Convention                           | Example                                                      | Why                                                          |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Folder/archive name includes version | `my-enterprise-app-1.4.2.tar.gz`                             | Multiple versions can sit side-by-side during upgrades; unambiguous in support tickets |
| Both `.tar.gz` and `.zip` shipped    | —                                                            | Covers Linux/Mac (`tar`) and Windows (no native `tar` in older environments) without customer needing extra tools |
| Consistent internal folder name      | archive extracts to `{product}-{version}/`, not a generic `dist/` or `output/` | Multiple extracted versions don't collide in the same parent directory |
| Docker image tag matches package version | `my-enterprise-app:1.4.2` (not `latest`)                  | Reproducible deploys; lets customers pin and roll back predictably |
