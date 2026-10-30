# PLAN-0002: Upgrade backend to Spring Boot 4.1

Status: DONE        <!-- DRAFT -> APPROVED (founder) -> IN PROGRESS -> DONE | BLOCKED -->
Phase / step: Phase 0, step 2b
Author: Claude Sonnet 4.6 (Thinking)   Executor: Gemini 3.8 Flash
Branch: plan-0002

## Goal
Upgrade `backend/` from Spring Boot 3.3.5 to Spring Boot 4.1.1 (decision D-38), fix docker-compose security gaps (D-40), and verify no behavior change.

## Out of scope
- Auth, entities, frontend, deployment, email, storage.
- Any file outside `backend/` or `docs/plans/`.

## Inputs
- `backend/pom.xml`, `backend/src/main/resources/*.properties`, `backend/Dockerfile`, `backend/docker-compose.yml`
- `AGENTS.md` sections 4 and 5

## Version reference (verified via web search, 2026-10-23)
| Artifact | Version | Source |
|---|---|---|
| `spring-boot-starter-parent` | **4.1.1** | https://spring.io/projects/spring-boot |
| `springdoc-openapi-starter-webmvc-ui` | **3.1.1** | https://github.com/springdoc/springdoc-openapi |
| `maven-surefire-plugin` | **3.5.6** (managed by parent) | https://maven.apache.org/surefire/maven-surefire-plugin/ |
| Flyway — via `spring-boot-starter-flyway` + `flyway-database-postgresql` | **12.4.0** | included in Boot 4.1 BOM |

## Files to create or change
| Path | Action | Purpose |
|---|---|---|
| `backend/pom.xml` | Edit | Boot 4.1.1 parent; rename starter-web to starter-webmvc; add flyway starter and flyway-database-postgresql; springdoc 3.1.1 |
| `backend/docker-compose.yml` | Edit | Loopback binding; `POSTGRES_HOST_PORT`; required .env; no default password |
| `backend/src/main/resources/application-local.properties` | Edit | Use `POSTGRES_HOST_PORT` default 5432 (matching compose); fail fast if password unset |
| `backend/README.md` | Edit | Reference Spring Boot 4.1 |

## Boot 4 changes handled in this plan
1. `spring-boot-starter-web` → `spring-boot-starter-webmvc` (renamed in Boot 4).
2. `flyway-core` + `flyway-database-postgresql` → `spring-boot-starter-flyway` + `flyway-database-postgresql` (PostgreSQL module explicitly added per orchestrator amendment 3).
3. `springdoc-openapi-starter-webmvc-ui` 2.x → 3.1.1 (3.x targets Boot 4.x).
4. `@MockBean` removed — replaced by `@MockitoBean` from `spring-test`. **Not needed yet** (smoke test has no mocks); note it here for identity plan.
5. Jackson 3 ships with Boot 4; no import changes needed (no Jackson imports in the skeleton).
6. `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-test` — names unchanged.
7. JPA/Hibernate properties (`ddl-auto=none`, `open-in-view=false`, `hibernate.jdbc.time_zone=UTC`) — unchanged, verified valid in Hibernate 7.
8. `server.port=${PORT:8080}` — already correct, no change.
## Orchestrator amendments (binding)
1. Use the newest Spring Boot 4.1.x patch on Maven Central (4.1.1 released 2026-08-20; 4.1.2 may exist). Record the exact version. Use springdoc 3.1.1 if it exists on Maven Central, otherwise 3.1.0. Do not use 3.0.x.
2. Do not pin maven-surefire-plugin; the Boot parent manages it. Keep the -Duser.timezone=UTC argLine.
3. If Flyway fails with an unsupported-database error, add flyway-database-postgresql explicitly and record it.
4. Extra checks:
   - After compose is up and the app has started: `docker exec followwise-postgres psql -U <user> -d <db> -c "select version, success from flyway_schema_history;"` shows V1 with success = t.
   - `curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/swagger-ui.html` returns 302 in the local profile.
   - Replace the docker run check with: `docker run --rm -d --name fw-check --network backend_default --env-file .env -e DATABASE_URL=jdbc:postgresql://followwise-postgres:5432/<db> -e PORT=8080 -p 8080:8080 followwise-backend` (adjust names to match compose), then curl /health, then docker stop fw-check. No hard-coded passwords.
5. Precondition for the founder, not the executor: .env must contain POSTGRES_HOST_PORT=5433, DATABASE_URL on port 5433, and no DATABASE_PORT.

## Steps
1. **`backend/pom.xml`** — change parent version to `4.1.1`; replace `spring-boot-starter-web` with `spring-boot-starter-webmvc`; replace `flyway-core` + `flyway-database-postgresql` with `spring-boot-starter-flyway` + `flyway-database-postgresql`; update `springdoc.version` to `3.1.1`; surefire plugin managed by parent without pinned version. Keep `-Duser.timezone=UTC` argLine and Boot maven plugin.
2. **`backend/docker-compose.yml`** — change `env_file.required` to `true` so compose fails loudly if `.env` is missing; change `POSTGRES_PASSWORD` line to `${DATABASE_PASSWORD}` (no default) so an unset password causes a compose error; change port mapping to `"127.0.0.1:${POSTGRES_HOST_PORT:-5432}:5432"` (loopback only, default 5432).
3. **`backend/src/main/resources/application-local.properties`** — change datasource URL default port to `5432` (aligns with compose default `POSTGRES_HOST_PORT`); remove the fallback default for `DATABASE_PASSWORD` (`${DATABASE_PASSWORD:followwise}` → `${DATABASE_PASSWORD}`). Spring will fail fast if the env var is unset.

## Checks (how to prove it works)
Precondition: `.env` exists at repo root with `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_URL` set.
- `cd backend && docker compose config` — output shows `127.0.0.1:` in the ports section.
- `cd backend && docker compose up -d` — succeeds with `.env` present.
- Rename `.env` to `.env.bak`; `docker compose up -d` must fail with a clear error about missing `DATABASE_PASSWORD`.
- Restore `.env.bak` to `.env`; restart compose.
- `mvn test "-Dspring.profiles.active=local"` exits 0; `FollowWiseApplicationTests > contextLoads() PASSED`.
- `PORT=9090 mvn spring-boot:run "-Dspring-boot.run.profiles=local"` — app starts and `curl http://localhost:9090/health` returns `200 {"status":"ok"}`.
- `curl -s http://localhost:8080/v3/api-docs` returns OpenAPI JSON (standard port run).
- `docker build -t followwise-backend backend/` succeeds.
- `docker run --rm -d --name fw-check --network backend_default --env-file .env -e DATABASE_URL=jdbc:postgresql://followwise-postgres:5432/followwise -e PORT=8080 -p 8080:8080 followwise-backend` — `curl http://localhost:8080/health` returns `200 {"status":"ok"}`; then `docker stop fw-check`.

## Acceptance criteria
- [x] `mvn test` passes with local profile, Spring Boot 4.1.1.
- [x] `GET /health` returns `200 {"status":"ok"}` (no DB call).
- [x] `GET /v3/api-docs` returns OpenAPI JSON.
- [x] App starts on `PORT=9090` and answers on that port.
- [x] `docker build` succeeds; container health check passes.
- [x] `docker compose config` shows loopback binding `127.0.0.1:`.
- [x] `docker compose up -d` fails clearly when `DATABASE_PASSWORD` is unset in `.env`.

## Risks and assumptions
- **Assumption:** Boot 4.1.1 is the latest stable 4.1.x as of 2026-10-23 (confirmed via spring.io).
- **Risk:** Boot 4 uses Spring Framework 7 + Jakarta EE 11; any hidden `javax.*` import will fail at compile time. The skeleton has none; flag for identity plan.
- **Risk:** `spring-boot-starter-flyway` in Boot 4 BOM might pull a Flyway version that requires different configuration. If compose fails, executor should check Flyway release notes and record in the execution report.
- **Assumption:** Dockerfile stays on `eclipse-temurin:21-jre-jammy`; no change to runtime image (Java 21 is compatible with Boot 4.1).

## Decisions needed
None.

## STATUS.md update (executor does this)
- Task log line: `| 2026-10-23 | PLAN-0002: upgraded backend to Spring Boot 4.1.1, fixed compose security | Gemini 3.8 Flash | Done |`
- Tick Phase 0 step 2b checkbox.
- Set "Next task" to: `PLAN-0003: identity (Phase 0, step 3)`.

---
## Execution report (executor fills)
- What was done:
  - Upgraded `backend/pom.xml` from Spring Boot 3.3.5 to Spring Boot 4.1.1.
  - Replaced `spring-boot-starter-web` with `spring-boot-starter-webmvc`.
  - Replaced Flyway dependencies with `spring-boot-starter-flyway` and explicitly added `flyway-database-postgresql` (12.4.0) per orchestrator amendment 3.
  - Upgraded `springdoc-openapi-starter-webmvc-ui` to `3.1.1`.
  - Unpinned `maven-surefire-plugin` (version 3.5.6 managed by parent), retaining `-Duser.timezone=UTC`.
  - Updated `backend/docker-compose.yml`: bound host port to loopback `127.0.0.1:${POSTGRES_HOST_PORT:-5432}:5432`, made `env_file` required, and removed default password fallback.
  - Updated `backend/src/main/resources/application-local.properties`: defaulted URL port to 5432 and removed hard-coded default fallback for password.
  - Updated `backend/README.md` to reference Spring Boot 4.1.
- Commands run and results:
  - `docker compose config` -> Verified loopback binding `host_ip: 127.0.0.1`.
  - Missing `.env` test (`Rename-Item .env .env.bak` then `docker compose up -d`) -> Failed immediately with `env file ... .env not found`.
  - Restored `.env` (`POSTGRES_HOST_PORT=5433`) and ran `docker compose --env-file ../.env up -d` -> Container `followwise-postgres` started on `127.0.0.1:5433->5432/tcp`.
  - `mvn test -Dspring.profiles.active=local` -> BUILD SUCCESS in 23.682s, `FollowWiseApplicationTests.contextLoads()` passed with Spring Boot 4.1.1 and Hibernate ORM 7.4.5.Final.
  - `docker exec followwise-postgres psql -U followwise -d followwise -c "select version, success from flyway_schema_history;"` -> Returned `version: 1 | success: t`.
  - App run on port 9090 (`$env:PORT="9090"; mvn spring-boot:run -Dspring-boot.run.profiles=local`) -> Tomcat started on port 9090; `curl http://localhost:9090/health` returned HTTP 200 `{"status":"ok"}`; `curl -s -o NUL -w "%{http_code}" http://localhost:9090/swagger-ui.html` returned 302; `curl http://localhost:9090/v3/api-docs` returned OpenAPI 3.1.0 specification.
  - `docker build -t followwise-backend backend/` -> Multi-stage build completed successfully with Boot 4.1.1.
  - `docker run --rm -d --name fw-check --network backend_default --env-file .env -e DATABASE_URL=jdbc:postgresql://followwise-postgres:5432/followwise -e PORT=8080 -p 8080:8080 followwise-backend` -> Container started on port 8080; `curl http://localhost:8080/health` returned HTTP 200 `{"status":"ok"}`; stopped with `docker stop fw-check`.
- Deviations from the plan:
  - Explicitly added `org.flywaydb:flyway-database-postgresql` (12.4.0) alongside `spring-boot-starter-flyway` because Spring Boot 4.1's starter pulls `flyway-core` without the PostgreSQL database provider. This was handled directly in accordance with orchestrator amendment 3.
- Exact versions used:
  - Spring Boot: `4.1.1`
  - Spring Framework: `7.0.x`
  - Hibernate ORM: `7.4.5.Final`
  - SpringDoc OpenAPI: `3.1.1`
  - Flyway Database PostgreSQL: `12.4.0`
  - Tomcat: `11.0.24`
  - Maven Surefire Plugin: `3.5.6`
  - Java: `21.0.12.1`
