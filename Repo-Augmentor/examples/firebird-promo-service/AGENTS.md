# Agent guide — firebird-promo-service

This document orients coding agents and contributors to **firebird-promo-service**: Chewy’s **Firebird** promotions orchestration layer (browse/eligibility/deals and checkout flows).

## What this repo is

- **Kotlin** on **Spring Boot 3.x** (see root `build.gradle` for the pinned Boot version).
- **Java 21 is required.** Gradle fails fast if the JVM is not 21 (`build.gradle` enforces `JavaVersion.VERSION_21`).
- Three Gradle modules:
  - **`firebird-common`** — shared library (no boot JAR): orchestration domain, `PromotionEngine`, services, persistence, Kafka, clients, most REST pieces used by both apps.
  - **`firebird-browse`** — deployable **browse** API (PDP/cart search, eligibility, SKU deals, promotion codes, scheduled jobs).
  - **`firebird-checkout`** — deployable **checkout** API (cart/order promotion orchestration, finalize paths, checkout-specific controllers and jobs).

Both apps share the `com.chewy.firebird.orchestrator` package and depend on `firebird-common`.

## Prerequisites

1. **JDK 21** — install and point `JAVA_HOME` (or use SDKMAN as in `README.md`).
2. **Artifactory** — `~/.gradle/gradle.properties` must define `artifactory_user`, `artifactory_password`, `artifactory_server` (and wrapper credentials if your org requires them). Without this, dependency resolution fails.
3. **Docker** — required for integration tests (`README.md`).
4. **Local secrets/config** — team-provided `application-dev.properties` / `application-stg.yaml` (see `README.md`); do not commit real credentials.

## Common commands

Run from the repository root:

| Task | Command |
|------|---------|
| Tests (+ per-subproject JaCoCo reports; root aggregates via `testCodeCoverageReport`) | `./gradlew test` |
| Full CI-style build | `./gradlew clean build` |
| Compile only | `./gradlew assemble` |
| Run browse (example profile) | `./gradlew :firebird-browse:bootRun --args='--spring.profiles.active=stg'` |
| Run checkout (example profile) | `./gradlew :firebird-checkout:bootRun --args='--spring.profiles.active=stg'` |

Reports (after tests):

- Test HTML: `build/reports/tests/test/index.html` (and per-module under each subproject).
- JaCoCo: subproject `build/reports/jacoco/test/html/index.html`; aggregated XML for Sonar is produced under `build/reports/jacoco/testCodeCoverageReport/` (see `build.gradle`).

## Where to work

| Concern | Typical location |
|--------|-------------------|
| Shared promotion logic, engine, rules, filters, most services | `firebird-common/src/main/kotlin/com/chewy/firebird/orchestrator/` |
| Browse-only controllers | `firebird-browse/src/main/kotlin/.../controller/` |
| Checkout-only controllers | `firebird-checkout/src/main/kotlin/.../controller/` |
| Auto-configuration registration | `META-INF/spring.factories` in each module that needs it |
| Infra / Kubernetes | `infra/helm/` |
| Architecture decisions | `docs/adrs/` |

Controllers are thin; heavy lifting lives in **`firebird-common`** (e.g. `engine/PromotionEngine.kt`, `service/`, `rules/`).

## Conventions for agents

- Match existing **Kotlin** style and Spring patterns in the same package.
- Prefer extending existing services and tests over parallel implementations.
- **Do not** commit secrets, real `application-*.yml` with credentials, or Artifactory keys.
- Swagger/OpenAPI and various internal Chewy libraries are on the classpath; follow existing client/config patterns when adding integrations.
- **AspectJ** load-time weaving may apply in runtime images; local agent setup is described in `README.md`.
- Large areas are **Sonar-excluded** from coverage metrics (see `sonarqube` `sonar.exclusions` in `build.gradle`); still add tests where behavior matters.

## Documentation map

- **Human onboarding & setup:** `README.md`
- **Architecture overview:** `docs/architecture.md`
- **ADRs:** `docs/adrs/` (e.g. `001-record-architecture-decision-records.md`)

## Escalation

Team/process questions (credentials, environments, Slack) belong with the **promo** / Firebird owners per `README.md` (`#promotions-public`, `@promo-dev-team`). This file stays technical and repo-local.
