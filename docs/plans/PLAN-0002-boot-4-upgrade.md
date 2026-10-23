# PLAN-0002: Upgrade backend to Spring Boot 4.1

Status: DRAFT        <!-- DRAFT -> APPROVED (founder) -> IN PROGRESS -> DONE | BLOCKED -->
Phase / step: Phase 0, step 2b
Author: Claude Sonnet 4.6 (Thinking)   Executor: (model that executed)
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
| `maven-surefire-plugin` | **3.6.0** | https://maven.apache.org/surefire/maven-surefire-plugin/ |
| Flyway — via `spring-boot-starter-flyway` | BOM-managed | included in Boot 4.1 BOM |

## Files to create or change
| Path | Action | Purpose |
|---|---|---|
| `backend/pom.xml` | Edit | Boot 4.1.1 parent; rename starter-web; use flyway starter; springdoc 3.1.1; surefire 3.6.0 |
| `backend/docker-compose.yml` | Edit | Loopback binding; `POSTGRES_HOST_PORT`; no default password |
| `backend/src/main/resources/application-local.properties` | Edit | Use `POSTGRES_HOST_PORT` default 5432 (matching compose) |

## Boot 4 changes handled in this plan
1. `spring-boot-starter-web` → `spring-boot-starter-webmvc` (renamed in Boot 4).
2. `flyway-core` + `flyway-database-postgresql` → `spring-boot-starter-flyway` (Boot 4 starter pulls both; BOM manages versions).
3. `springdoc-openapi-starter-webmvc-ui` 2.x → 3.1.1 (3.x targets Boot 4.x).
4. `@MockBean` removed — replaced by `@MockitoBean` from `spring-test`. **Not needed yet** (smoke test has no mocks); note it here for identity plan.
5. Jackson 3 ships with Boot 4; no import changes needed (no Jackson imports in the skeleton).
6. `spring-boot-starter-data-jpa`, `spring-boot-starter-validation`, `spring-boot-starter-test` — names unchanged.
7. JPA/Hibernate properties (`ddl-auto=none`, `open-in-view=false`, `hibernate.jdbc.time_zone=UTC`) — unchanged, verified valid in Hibernate 7.
8. `server.port=${PORT:8080}` — already correct, no change.

## Steps
1. **`backend/pom.xml`** — change parent version to `4.1.1`; replace `spring-boot-starter-web` with `spring-boot-starter-webmvc`; replace `flyway-core` + `flyway-database-postgresql` with `spring-boot-starter-flyway`; update `springdoc.version` to `3.1.1`; add explicit surefire version `3.6.0`. Keep `-Duser.timezone=UTC` argLine and Boot maven plugin (no version tag — inherits parent).
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
- `docker run --rm -d --name fw-check --network backend_default -e PORT=8080 -e DATABASE_URL=jdbc:postgresql://followwise-postgres:5432/followwise -e DATABASE_USER=followwise -e DATABASE_PASSWORD=followwise -p 8080:8080 followwise-backend` — `curl http://localhost:8080/health` returns `200 {"status":"ok"}`; then `docker stop fw-check`.

## Acceptance criteria
- [ ] `mvn test` passes with local profile, Spring Boot 4.1.1.
- [ ] `GET /health` returns `200 {"status":"ok"}` (no DB call).
- [ ] `GET /v3/api-docs` returns OpenAPI JSON.
- [ ] App starts on `PORT=9090` and answers on that port.
- [ ] `docker build` succeeds; container health check passes.
- [ ] `docker compose config` shows loopback binding `127.0.0.1:`.
- [ ] `docker compose up -d` fails clearly when `DATABASE_PASSWORD` is unset in `.env`.

## Risks and assumptions
- **Assumption:** Boot 4.1.1 is the latest stable 4.1.x as of 2026-10-23 (confirmed via spring.io).
- **Risk:** Boot 4 uses Spring Framework 7 + Jakarta EE 11; any hidden `javax.*` import will fail at compile time. The skeleton has none; flag for identity plan.
- **Risk:** `spring-boot-starter-flyway` in Boot 4 BOM might pull a Flyway version that requires different configuration. If compose fails, executor should check Flyway release notes and record in the execution report.
- **Assumption:** Dockerfile stays on `eclipse-temurin:21-jre-jammy`; no change to runtime image (Java 21 is compatible with Boot 4.1).

## Decisions needed
None.

## STATUS.md update (executor does this)
- Task log line: `| <actual date> | PLAN-0002: upgraded backend to Spring Boot 4.1.1, fixed compose security | <model> | Done |`
- Tick Phase 0 step 2b checkbox.
- Set "Next task" to: `PLAN-0003: identity (Phase 0, step 3)`.

---
## Execution report (executor fills)
- What was done:
- Commands run and results:
- Deviations from the plan:
- Follow-ups or new risks:
