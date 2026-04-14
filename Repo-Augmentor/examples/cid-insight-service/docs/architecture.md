# Project Architecture Blueprint

**Scope:** **cid-insight-service** (Customer Insights API; often referred to as “CID insights”) and **cid-ingestion** (event pipeline; not in this repo—inferred from shared config).  
**Generated:** April 2026  
**Stack (insight service):** Java 17, Spring Boot 2.7.8, Spring WebFlux, Spring GraphQL, Federation router, `com.chewy.cids:common` 0.428.0

---

## 1. Architecture overview

### 1.1 System roles

| System | Role | Primary I/O |
|--------|------|-------------|
| **cid-insight-service** | Read-optimized API: GraphQL, REST, MCP, Apollo Federation subgraph | **Reads** OpenSearch, DynamoDB, Neptune; **calls** downstream HTTP services; **writes** Kafka only for MFA challenge flow |
| **cid-ingestion** | Event-driven indexer (separate codebase) | **Consumes** Kafka; **writes** OpenSearch, DynamoDB, Neptune |

### 1.2 Architectural pattern

- **Monolithic Spring Boot** service with **layered** organization: controllers → services → clients/DAOs.
- **Reactive** stack (`WebFlux`, reactive clients) for I/O-bound workloads.
- **Integration** with **chewy-api-router** via Federation (`POST /router`).
- **Shared platform** concerns delegated to **`cids:common`** (clients, config, history abstractions).

### 1.3 Guiding principles (evident in code)

1. **Read/write split:** Heavy indexing is **not** done in insight service; **cid-ingestion** owns bulk writes to search stores.
2. **Single Kafka produce path** from insight service for **MFA challenge** → topic → ingestion → OpenSearch.
3. **OAuth2 resource server** on most APIs; **health, router, GraphiQL, Swagger, MCP** explicitly open.
4. **Resilience4j** for selected downstream paths (e.g. Kyrios circuit breaker / retry in `application.yml`).
5. **Secrets:** AWS Secrets Manager bootstrap via `CommonUtils.injectSharedSecrets()` at application start.

---

## 2. Architecture visualization (conceptual)

### 2.1 E2E with cid-ingestion

Sources and external producers feed **Kafka (MSK)**. **cid-ingestion** consumes, transforms, and indexes **OpenSearch**, **DynamoDB**, and **Neptune**. **cid-insight-service** reads those stores and optional HTTP downstreams; it **produces** to Kafka only for **MFA** data path.

See also:

- `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.md` (narrative + tables)
- `docs/E2E_ARCHITECTURE_diagrams.mmd` (Mermaid for tools)
- `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.xml` (Draw.io / Lucid import)

### 2.2 C4-style (container level)

```text
[Clients: UI, Agents, chewy-api-router]
        │
        ▼
┌───────────────────────────────┐
│     cid-insight-service       │
│  GraphQL │ REST │ MCP │ /router│
└───────────────────────────────┘
        │                    │
        │ read               │ produce (MFA only)
        ▼                    ▼
 OpenSearch / DDB / Neptune   Kafka (MSK)
        ▲                    │
        │ consume            │
        └────────────────────┘
        cid-ingestion
```

---

## 3. cid-insight-service — structural map

### 3.1 Package layout (main)

| Area | Package | Responsibility |
|------|---------|----------------|
| Bootstrap | `...CustomerInsightServiceApplication` | Spring Boot entry; shared secrets injection |
| Config | `...config` | Security, GraphQL, downstream imports, MCP, Swagger, dataloaders, history clients |
| API — REST / GraphQL entry | `...controller`, `...controller.v2`, `...controller.v2.insight` | Spring `@RestController` + GraphQL controllers; filters under `controller.filter` |
| Domain services | `...service`, `...service.v2`, `...service.v2.insight` | Business orchestration, mapping to GraphQL types |
| Integration | Uses `com.chewy.cids.*` | OpenSearch history clients, DynamoDB DAOs, Kafka publishing, Neptune Gremlin |
| Cross-cutting | `...context`, `...exception`, `...constants` | Request context, GraphQL/HTTP errors, scopes and paths |

### 3.2 Entry points (implemented)

| Type | Path / mechanism | Notes |
|------|------------------|-------|
| GraphQL | `POST /graphql` | Primary schema + GraphiQL when enabled |
| Federation | `POST /router` | Subgraph for router; `RouterSettings` + allowlist in `application.yml` |
| MCP | `GET/POST /mcp`, `/mcp/**` | Optional; `cid.mcp.enabled` (default true) gates `McpConfig` / `CidInsightMcpServer` |
| REST | `/api/v1/...` | Legacy v1 + device/MFA write |
| Actuator | `/actuator/*` | Health exposed; deeper actuator paths JWT-protected |

### 3.3 Dependency flow (layers)

1. **Controller** resolves arguments, applies auth context (filters), delegates to **service**.
2. **Service** composes **cids** clients (OpenSearch reactive/history), **DynamoDB** enhanced clients, **Neptune** Gremlin, or **WebClient**-based downstream services from `DownstreamServiceConfig`.
3. **Config** wires `DownstreamSettings`, `WebClientBuilderConfig`, and many `*ServiceImpl` classes from `cids:common`.

### 3.4 Data architecture (read)

| Concern | Mechanism |
|---------|-----------|
| OpenSearch | Aliases per domain: `customer_insights`, `history_incident`, `history_oms`, `history_interaction`, `history_account`, `customer_composite` (see `application.yml` `cid.opensearch.indices`) |
| DynamoDB | Tables via `cid.dynamodb.cidTablePrefix`; DAOs for customer, person, pet, identity-mapping |
| Neptune | Gremlin client; identity graph queries |
| Cache | Caffeine (`cache.ttlMinutes`) |
| Downstream HTTP | Order, Returns, Kyrios, GiftCard, Promotions, Pet Health, etc. (`downstream.*` in `application.yml`) |

### 3.5 Data architecture (write path in this service)

- **MFA challenge:** `MfaChallengeController` → `MfaChallengeHistoryClient` → **Kafka** → **cid-ingestion** → OpenSearch index `history_mfa_challenge_${AWS_SHORTREGION}`.

---

## 4. cid-ingestion (inferred — not in repo)

**Source of inference:** Kafka secret path `AmazonMSK_common_{env}_{region}_com.chewy.csbb.cid-ingestion`, index names in config, and E2E doc.

### 4.1 Responsibilities

1. Consume CID-related **Kafka** topics (orders, incidents, device, MFA, etc.).
2. Transform / enrich events.
3. Bulk **index** OpenSearch; upsert **DynamoDB**; mutate **Neptune** graph.

### 4.2 Contract with cid-insight-service

- **Schema alignment:** OpenSearch mappings and DynamoDB key design must match what insight clients expect.
- **Topic contracts:** Producers (including insight service for MFA) and consumers must agree on payload format.
- **Region suffix:** `AWS_SHORTREGION` (e.g. `use1`) participates in index naming for device/MFA history.

---

## 5. Cross-cutting concerns

### 5.1 Authentication and authorization

- **OAuth2 JWT** resource server (`issuer-uri` / `jwk-set-uri` from `CHEWY_S2S_AUTH_ISSUER_URI`).
- **Scopes** (see `ScopeConstants`): e.g. `v1_insight_read`, `v2_insight_read`, `v1_device_data_write`.
- **Public chains** (`SecurityConfig` `@Order(0)`): health, info, Swagger, GraphiQL, **`/router`**, **`/mcp`** GET/POST.
- **Auth context / token validation** via Chewy auth libraries + `AwsSettings` region and `spring.environment`.

### 5.2 Configuration management

- **YAML:** `application.yml` + optional `spring.config.additional-location` (e.g. `classpath:/cids-common.yml`).
- **Env-specific overrides:** `etc/*_env_info.txt` + `etc/run_env.sh` for local bootRun.
- **Secrets:** AWS Secrets Manager paths for OpenSearch, Kafka, shared CID secret (`CIDS_SHARED_SECRET` pattern in run script).

### 5.3 Observability

- **Actuator:** health, info, metrics, mappings, etc.
- **Logging:** Logback JSON; structured `%X` MDC-style fields in pattern.
- **Tracing:** Spring Cloud Sleuth on classpath.
- **Datadog:** Agent JAR referenced in `build.gradle` docker/JVM args for deployment images.

### 5.4 Error handling

- GraphQL: `GraphQlExceptionResolver` and related resolvers under `exception`.
- HTTP: `SpringExceptionResolversExceptionHandler`, domain exceptions (`CustomerNotFoundException`, etc.).

### 5.5 Resilience

- **Resilience4j** instances in `application.yml` (e.g. Kyrios circuit breaker and retry).
- **Timeouts** centralized under `cids.common.timeout` for HTTP, DynamoDB, Kafka producer, OpenSearch, Neptune.

---

## 6. Service communication patterns

| Pattern | Usage |
|---------|--------|
| Synchronous HTTP | WebFlux `WebClient` to Chewy platform services (orders, returns, loyalty, …) |
| Synchronous-ish search | OpenSearch via `cids` reactive/history clients |
| Graph DB | Neptune Gremlin, async scheduling pool in config |
| Async event | Kafka producer for MFA path only (in this service) |
| Federation | HTTP POST `/router` for subgraph operations allowlisted in config |

---

## 7. Technology-specific notes (Java / Spring)

- **Boot 2.7** + **Spring Framework 5.3** — note for **MCP**: newer MCP Spring modules may target Spring 6; validate runtime compatibility before enabling integration tests or upgrading.
- **DGS codegen** — GraphQL types generated under `build/generated/sources/dgs-codegen`; `generateJava` task in `build.gradle`.
- **Lombok + MapStruct** — DTO mapping in `mapper` packages.
- **Neptune / Janus / Gremlin** — client and scheduler threads configured under `cid.neptune` and `cid.scheduler.neptune`.

---

## 8. Testing architecture

| Style | Examples |
|-------|----------|
| Unit / slice | Controller tests with `@WebFluxTest` / mocked services |
| Integration | `@SpringBootTest` with `WebTestClient`, heavy use of `@MockBean` for AWS/OpenSearch/Kafka |
| GraphQL context | `GraphQLContextIntegrationTest` — selective imports; `cid.mcp.enabled=false` to avoid MCP classpath issues |

**Note:** `McpServerIntegrationTest` may be excluded or disabled in Gradle due to Spring/MCP version alignment — see `build.gradle` `test { exclude ... }` if present.

---

## 9. Deployment architecture

- **Container:** Docker / Helm under `infra/helm`; image repo pattern `csbb/cid-insight-service` per `build.gradle`.
- **Runtime:** EKS-style deployment; env-specific values under `infra/helm/values*`.
- **Local:** `etc/run_env.sh <env>` with AWS profile and shared secret path.

---

## 10. Extension guidelines (new features)

1. **New GraphQL operation:** Add/update schema in `etc/graphql`, run codegen, implement controller under `controller/v2` (or insight subpackage) and service under `service/v2`.
2. **New read datasource:** Prefer extending through **`cids:common`** clients or new Spring `@Configuration` beans; keep controllers thin.
3. **New Kafka producer path:** Rare — align with **cid-ingestion** team on topic, schema, and index before adding producers in insight service.
4. **New public route:** Update `SecurityConfig` no-auth chain if the path must bypass JWT.

---

## 11. Architectural decision records (implicit)

| Topic | Decision observable in codebase |
|-------|--------------------------------|
| Write vs read | Insight service **reads** canonical indexes; **ingestion** **writes** after Kafka |
| WebFlux | Reactive pipeline for I/O concurrency |
| Federation | Single `/router` subgraph with explicit allowlist |
| MCP | Exposed without JWT for agent tooling; tool executes GraphQL in-process |
| Shared library | Heavy reliance on **`cids:common`** reduces duplication but couples release versions |

---

## 12. Blueprint maintenance

- Regenerate or refresh this document when **major** changes occur: new storage, new Kafka contracts, router/Federation shape, or breaking `cids:common` upgrades.
- Keep **E2E** narrative authoritative for **cross-service** flows: `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.md`.
- **Repository:** `cid-insight-service` (this repo). **cid-ingestion:** separate repo — confirm behaviors with that team before changing ingestion assumptions.

---

## 13. References

| Artifact | Path |
|----------|------|
| E2E architecture | `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.md` |
| Mermaid pack | `docs/E2E_ARCHITECTURE_diagrams.mmd` |
| Draw.io / Lucid | `docs/E2E_ARCHITECTURE_cid-insight-service_cid-ingestion.xml` |
| Main config | `src/main/resources/application.yml` |
| Security | `src/main/java/.../config/SecurityConfig.java` |
| Downstream wiring | `src/main/java/.../config/DownstreamServiceConfig.java` |
