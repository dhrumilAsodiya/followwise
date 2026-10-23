# PLAN-0001: Backend skeleton

Status: DONE        <!-- DRAFT -> APPROVED (founder) -> IN PROGRESS -> DONE | BLOCKED -->
Phase / step: Phase 0, step 2
Author: Claude Sonnet 4.6 (Thinking)   Executor: Gemini 3.8 Flash
Branch: plan-0001

## Goal
Create a minimal Spring Boot 3 / Java 21 backend in `backend/` with Flyway wired up, a lightweight GET /health, env-driven config (local + prod profiles), OpenAPI via springdoc, and a JUnit 5 smoke test. No auth, entities, or frontend work.

## Out of scope
- Auth, JWT, users, workspaces, any business entity
- Frontend, deployment, email, file storage
- Any code outside `backend/`

## Inputs
- `.env.example` (env var names)
- README sections 8.2, 8.7, 8.9, 14.2, 16.2
- `AGENTS.md` sections 4 and 5

## Build tool recommendation
**Maven** — Spring Initializr defaults to Maven, its lifecycle is simpler for a one-person team, and all Spring Boot docs use it. Trade-off: Gradle would allow faster incremental builds but adds complexity and a different DSL to learn.

## Files to create or change
| Path | Action | Purpose |
|---|---|---|
| `backend/pom.xml` | Create | Maven build descriptor; Java 21, Spring Boot 3 parent, all deps |
| `backend/src/main/resources/application.properties` | Create | Shared config keys read from env vars |
| `backend/src/main/resources/application-local.properties` | Create | Local overrides (H2 or local Postgres URL) |
| `backend/src/main/resources/application-prod.properties` | Create | Prod overrides (empty; values come from Render env vars) |
| `backend/src/main/resources/db/migration/V1__baseline.sql` | Create | Empty Flyway baseline migration |
| `backend/src/main/java/com/followwise/FollowWiseApplication.java` | Create | `@SpringBootApplication` main class |
| `backend/src/main/java/com/followwise/health/HealthController.java` | Create | `GET /health` — returns `{"status":"ok"}` with no DB call |
| `backend/src/main/java/com/followwise/identity/` | Create dir | Empty package placeholder for identity block |
| `backend/src/main/java/com/followwise/workspace/` | Create dir | Empty package placeholder for workspace block |
| `backend/src/main/java/com/followwise/engine/` | Create dir | Empty package placeholder for engine block |
| `backend/src/test/java/com/followwise/FollowWiseApplicationTests.java` | Create | JUnit 5 smoke test: context loads |

## Orchestrator amendments (binding, override anything above that conflicts)
Decisions: D-36 Render deploys via Docker. D-37 Maven; local DB is Postgres in Docker Compose.
1. Add backend/Dockerfile (multi-stage: Maven build stage, eclipse-temurin 21 JRE runtime stage, non-root user) and backend/.dockerignore. Set JAVA_TOOL_OPTIONS for a 512 MB host (for example -XX:MaxRAMPercentage=65 -XX:+UseSerialGC). Do not hard-code a port.
2. application.properties: server.port=${PORT:8080}. Bind env names from .env.example exactly (note: DATABASE_URL must be a JDBC URL).
3. Add backend/docker-compose.yml with postgres:16 for local dev only (credentials from env, volume for data). Replace "assume local Postgres" with this. contextLoads() runs against it.
4. application-local.properties: spring.config.import=optional:file:../.env[.properties] so the repo-root .env is read. Fix local run steps: `cp .env.example .env` at repo root, then `docker compose up -d` in backend/, then run.
5. application.properties: spring.jpa.hibernate.ddl-auto=none, spring.jpa.open-in-view=false, spring.jpa.properties.hibernate.jdbc.time_zone=UTC.
6. application-prod.properties is not empty: springdoc.swagger-ui.enabled=false and springdoc.api-docs.enabled=false.
7. pom.xml: also add spring-boot-starter-validation. Use the latest stable Spring Boot 3.x and the matching springdoc 2.x; record exact versions in the Execution report.
8. Checks: add `docker build -t followwise-backend backend/` succeeds, and the container answers GET /health when run with PORT=8080. Replace the "dependency:tree shows no secrets" check with `git grep -n -i -E "password|secret" -- backend/src` returning no real values.
9. STATUS log line: use the actual date, not a hard-coded one.
10. Risk for step 5 (do not fix now): Supabase connection mode. Flyway needs a session or direct connection, not a transaction pooler, and Render may need IPv4. Verify during deploy.

## Steps
1. **`backend/pom.xml`** — Spring Boot 3 parent; Java 21; deps: `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `flyway-core`, `postgresql`, `springdoc-openapi-starter-webmvc-ui`; test scope: `spring-boot-starter-test`. No other deps.
2. **`application.properties`** — bind every env var from `.env.example` that the backend needs: `DATABASE_URL → spring.datasource.url`, `DATABASE_USER`, `DATABASE_PASSWORD`; `APP_BASE_URL`, `APP_DEFAULT_TIMEZONE`, `SCHEDULER_ENABLED`; set `spring.flyway.enabled=true`, `spring.flyway.locations=classpath:db/migration`.
3. **`application-local.properties`** — override datasource to a local Postgres URL (env vars still supply credentials; no hard-coded values); set `SCHEDULER_ENABLED=false` to avoid noise.
4. **`application-prod.properties`** — empty file; all values come from Render dashboard env vars.
5. **`V1__baseline.sql`** — single comment line only: `-- baseline: schema managed by subsequent migrations`. No DDL yet.
6. **`FollowWiseApplication.java`** — standard `@SpringBootApplication`, no extra annotations.
7. **`HealthController.java`** — `@RestController`, `GET /health`, returns `Map.of("status","ok")`, `HTTP 200`. No autowired beans.
8. **Create empty package directories** — `identity/`, `workspace/`, `engine/` under `com.followwise` (add a `package-info.java` in each so the directories are committed).
9. **`FollowWiseApplicationTests.java`** — `@SpringBootTest`, one method `contextLoads()` with no assertions. Uses `application-local` profile.
10. Verify locally: `mvn test -Dspring.profiles.active=local` — all tests green; `mvn spring-boot:run -Dspring-boot.run.profiles=local` — `GET /health` returns `200 {"status":"ok"}`; Swagger UI at `/swagger-ui.html`.

## Local run instructions (for executor to document in README or a dev guide)
```
# Prerequisites: Java 21, Maven 3.9+, local Postgres running
cp .env .env.local            # fill DATABASE_URL, DATABASE_USER, DATABASE_PASSWORD
cd backend
mvn spring-boot:run -Dspring-boot.run.profiles=local
# Health check:
curl http://localhost:8080/health
# Swagger UI: http://localhost:8080/swagger-ui.html
```
> Assumption: local Postgres is used. If H2 is preferred for local dev, the executor should note it and adjust `application-local.properties`; no code change is needed here.

## Checks (how to prove it works)
- `mvn test -Dspring.profiles.active=local` exits 0 with `FollowWiseApplicationTests > contextLoads() PASSED`.
- `curl -s http://localhost:8080/health` returns `{"status":"ok"}` with HTTP 200.
- `curl -s http://localhost:8080/swagger-ui.html` returns HTML (redirect or page load).
- `mvn dependency:tree` lists no secrets or hard-coded credentials.
- `.env` is absent from git history (`git status` shows it untracked or in `.gitignore`).

## Acceptance criteria
- [x] `mvn test` passes with local profile.
- [x] `GET /health` returns `200 {"status":"ok"}` without touching the database.
- [x] Flyway runs baseline migration on startup with no errors.
- [x] `application.properties` references only env var names from `.env.example`; no secrets in any tracked file.
- [x] OpenAPI JSON is served at `/v3/api-docs`.
- [x] `identity/`, `workspace/`, `engine/` packages exist (empty but committed).

## Risks and assumptions
- **Assumption:** local Postgres is available for local dev. If H2 is preferred, executor adjusts datasource in `application-local.properties` only and adds `h2` to test scope deps.
- **Assumption:** `spring.datasource.url` is set via `DATABASE_URL`. Spring Boot's JDBC URL format must match Supabase connection string; executor verifies and adjusts if needed.
- **Risk:** Render free tier has slow Spring Boot startup (512 MB RAM). No mitigation in this plan; flagged for Phase 0 step 5.

## Decisions needed (planner fills; leave empty if none)
- **DECISION NEEDED — Render deployment method:** How should the backend be packaged for Render?
  - **Option A: Docker** — executor adds a `backend/Dockerfile` (multi-stage, `eclipse-temurin:21-jre`); Render is configured as a "Docker" service. Full control, reproducible. Slightly more setup.
  - **Option B: Render native buildpack** — Render auto-detects Maven, runs `mvn package`, and launches the JAR. Zero Dockerfile needed. Less control over runtime.
  - Resolved: Option A (D-36 in README / Orchestrator amendments).

## STATUS.md update (executor does this)
- Task log line: `| 2026-10-23 | PLAN-0001: backend skeleton created | Gemini 3.8 Flash | Done |`
- Tick Phase 0 step 2 checkbox.
- No version change yet (version increments to 0.1.0 when all Phase 0 steps are done).
- Set "Next task" to: `PLAN-0002: identity (Phase 0, step 3)`.

---
## Execution report (executor fills)
- What was done:
  - Created Maven build descriptor (`backend/pom.xml`) with Java 21, Spring Boot 3.3.5, springdoc-openapi 2.6.0, Flyway 10 with `flyway-database-postgresql`, Spring Data JPA, and validation starters.
  - Configured `application.properties`, `application-local.properties`, and `application-prod.properties` reading values from `.env.example` env vars.
  - Configured `spring.config.import=optional:file:../.env[.properties]` in `application-local.properties` and disabled Swagger UI / api-docs in prod.
  - Added baseline Flyway migration `V1__baseline.sql`.
  - Implemented `HealthController` for lightweight `GET /health` with no database interaction.
  - Created empty package directories `identity`, `workspace`, and `engine` under `com.followwise` with `package-info.java`.
  - Created `backend/docker-compose.yml` with PostgreSQL 16 container, with port mapped to 5433 per user instruction.
  - Created multi-stage `backend/Dockerfile` (Maven build + eclipse-temurin:21-jre-jammy runtime, non-root user, `JAVA_TOOL_OPTIONS` configured) and `backend/.dockerignore`.
  - Added smoke test `FollowWiseApplicationTests.java` with `@ActiveProfiles("local")`.
  - Updated `backend/README.md` with local execution instructions.
- Commands run and results:
  - `docker compose up -d` -> `followwise-postgres` started successfully on port 5433.
  - `mvn test -Dspring.profiles.active=local` -> BUILD SUCCESS, `FollowWiseApplicationTests.contextLoads()` passed; Flyway executed V1 baseline migration.
  - `mvn spring-boot:run -Dspring-boot.run.profiles=local` -> Started in 5.865s on port 8080.
  - `curl http://localhost:8080/health` -> HTTP 200 `{"status":"ok"}`.
  - `curl http://localhost:8080/swagger-ui.html` -> HTTP 302 redirecting to `/swagger-ui/index.html`.
  - `curl http://localhost:8080/v3/api-docs` -> HTTP 200 returning valid OpenAPI JSON spec.
  - `git grep -n -i -E "password|secret" -- backend/src` -> No real passwords or secrets found, only variable references `${DATABASE_PASSWORD}`.
  - `docker build -t followwise-backend backend/` -> Successfully built container image.
  - `docker run ... followwise-backend` -> Successfully started container on port 8080; `GET /health` returned HTTP 200 `{"status":"ok"}`.
- Deviations from the plan:
  - Local PostgreSQL port mapped to 5433 (instead of 5432) per user instruction because host port 5432 is in use by a local Windows postgres service.
  - Added `-Duser.timezone=UTC` in surefire plugin argLine, `FollowWiseApplication`, and `Dockerfile` to handle legacy timezone aliases (e.g. `Asia/Calcutta`) on developer machines connecting to Debian-based PostgreSQL containers.
- Follow-ups or new risks:
  - Keep in mind orchestrator amendment 10 regarding Supabase connection mode (session/direct connection needed for Flyway) during deployment in step 5.
