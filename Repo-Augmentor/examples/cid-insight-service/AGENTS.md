# AGENTS.md — cid-insight-service

Guidance for AI coding agents working in **Customer Insights Data Service** (`cid-insight-service`).

## What this is

- **Stack:** Java 17, Spring Boot 2.7.8, Spring WebFlux, Spring GraphQL, Apollo Federation (`/router`), OAuth2 JWT.
- **Role:** Read-heavy API for customer insights (GraphQL, REST, MCP). Heavy indexing is done by **cid-ingestion** via Kafka; this service mostly **reads** OpenSearch, DynamoDB, Neptune and calls downstream HTTP APIs.
- **Shared lib:** `com.chewy.cids:common` (and `cidsModel`, `cidInsightModel`) — config, history clients, Kafka publish (MFA path), etc.

## Common commands

| Task | Command |
|------|---------|
| Build | `./gradlew build` |
| Tests | `./gradlew test` |
| Format (apply) | `./gradlew format` |
| Format check | `./gradlew checkFormat` |
| Run locally (dev) | `etc/run_env.sh dev` — requires valid AWS profile (e.g. `aws sso login --profile development-nebula`) and Artifactory credentials in Gradle |
| Boot JAR | `./gradlew bootJar` |
| Docker | `./gradlew bootJar -x test` then `docker compose up --build` (see `README` / `docker-compose.yml`) |

## Project layout

- `src/main/java/com/chewy/cid/insight/` — application code: `config`, `controller`, `controller.v2`, `service`, `mcp`, `mapper`, `graphql`, `exception`, `context`.
- `src/main/resources/application.yml` — main config; `CHEWY_S2S_AUTH_ISSUER_URI` required at runtime.
- `etc/graphql/` — GraphQL schema inputs for **DGS codegen** (`generateJava` in `build.gradle` → `build/generated/sources/dgs-codegen`).
- `etc/*_env_info.txt`, `etc/run_env.sh` — per-environment boot args for local run.
- `docs/` — architecture docs (e.g. `Project_Architecture_Blueprint.md`, E2E with cid-ingestion).
- `infra/helm/` — deployment values.

## Conventions

1. **Formatting:** spring-javaformat; pre-commit hook via `etc/GitHook-pre-commit`. Do not bypass `checkFormat` in CI-minded workflows.
2. **New GraphQL:** Update schema under `etc/graphql/`, run `./gradlew compileJava` (or build) to regenerate types; implement controllers under `controller.v2` and services under `service.v2`.
3. **Security:** Most routes require JWT; `SecurityConfig` permits health, swagger, graphiql, `/router`, `/mcp` without auth. Follow existing patterns for new public paths.
4. **MCP:** `cid.mcp.enabled` (default true) gates `McpConfig` / `CidInsightMcpServer`. Some integration tests set `cid.mcp.enabled=false` to avoid loading MCP beans. MCP Spring SDK may lag Spring Boot 2.7 — verify compatibility when changing MCP tests.
5. **Dependencies:** Build needs Maven (Artifactory) credentials in `gradle.properties` or env; do not commit secrets.
6. **Coverage:** JaCoCo in `jacoco.gradle`; Sonar for CI; prefer not lowering `codeCoverageThreshold` in `gradle.properties`.

## Cross-service context

- **cid-ingestion** consumes Kafka and writes OpenSearch/DynamoDB/Neptune; insight service reads the same stores. See `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.md`.
- Kafka secret path naming ties to ingestion: `AmazonMSK_*_com.chewy.csbb.cid-ingestion` (see `application.yml`).

## Testing notes

- Full-context tests use `@SpringBootTest` + many `@MockBean` entries for AWS/OpenSearch/Kafka — mirror patterns in `GraphQLContextIntegrationTest` when adding similar tests.
- Prefer focused unit/slice tests where possible to keep startup cost down.

## OpenCode /init

If you use **OpenCode**, connect a provider first (`opencode auth login`), then run **`/init`** in the TUI or **`POST /session/:id/init`** on **`opencode serve`** to let the model refresh this file from the live tree. This file was seeded manually so the repo has agent context **without** requiring API keys in the agent environment.
