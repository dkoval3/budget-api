# Implementation Plan: Platform Skeleton

**Branch**: `001-platform-skeleton` | **Date**: 2026-10-12 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-platform-skeleton/spec.md`

## Summary

Turn the empty Maven project into a runnable, gated, observable Spring Boot 4.1 / WebFlux / R2DBC
service with an empty public contract, ready for the Users and auth spec to add endpoints.

The skeleton delivers:

- the constitution's layer packages and ArchUnit rules;
- contract-first generation from two OpenAPI 3.1 files (public and internal), with lint, a
  breaking-change check, contract validation in tests, and release publishing;
- Problem Details, `X-Request-Id` and `Cache-Control: no-store` on every response;
- a Cognito-style resource server with separate user and service chains;
- shared owner-scoping, version-check and create-with-ID helpers, proven through a test-only
  record;
- two database roles (owner for Flyway, app for requests);
- validated per-environment config, with AppConfig and Secrets Manager imports through Spring
  Cloud AWS;
- JSON logs, OTel traces and metrics, and a log privacy test;
- a Dockerfile, and a Compose stack with PostgreSQL 17, a mock OIDC server, the API, the UI
  (once released) and a guarded dev seed;
- GitHub Actions workflows for verify, image scan with E2E, contract publishing and scheduled
  scans.

AWS provisioning and deployment are deferred (spec, option C). Technical decisions are in
[research.md](research.md).

## Technical Context

**Language/Version**: Java 25

**Primary Dependencies**:

- Spring Boot 4.1.x (`spring-boot-starter-parent`): WebFlux (Netty), Security OAuth2 Resource
  Server, Data R2DBC (`r2dbc-postgresql`, `r2dbc-pool`), Flyway (+ PostgreSQL JDBC driver,
  migrations only), Validation, Actuator, OpenTelemetry starter (Micrometer Tracing OTel bridge,
  OTLP registry).
- `openapi-generator-maven-plugin`.
- Spring Cloud AWS 4.2.x: Secrets Manager and AppConfig starters (R11).
- JSpecify.

**Storage**: PostgreSQL 17, with schema `budgit`, roles `budgit_owner` (migrations) and
`budgit_app` (requests). There are no business tables in this feature.

**Testing**:

- JUnit 6, AssertJ, Reactor `StepVerifier`, `WebTestClient`.
- Testcontainers (Postgres 17, mock-oauth2-server).
- BlockHound (spike R15).
- Contract validation: `openapi-request-validator-core` 3.x behind an `ExchangeFilterFunction`
  (R4).
- ArchUnit, JaCoCo.
- WireMock (AWS loader tests).

**Build gates in `./mvnw verify`**: Spotless (Palantir), Error Prone + NullAway, Redocly lint
and oasdiff (via Docker), OWASP dependency-check.

**Target Platform**: Linux container (Temurin 25 JRE). Runs locally under Docker Compose; AWS ECS
or EKS later (deferred).

**Project Type**: Web service (single Maven module, REST/JSON API).

**Performance Goals**:

- No business endpoints yet, so API §X latency targets do not apply.
- Startup to ready in ≤ 60 s against a reachable database (SC-010).
- Full PR verification in < 15 min (SC-009).

**Constraints**:

- Non-blocking request path (API §II).
- No `localhost` outside `local`/`test` except declared sidecar endpoints; no non-Cognito issuer
  outside `local`/`test` (API §VI v1.1.0).
- No financial or personal data in logs, metrics or traces.
- Docker is required to build (Testcontainers, lint and diff images).

**Scale/Scope**: Single user (the owner) for the MVP. About 40–60 production classes and about
25 test classes. 3 GitHub workflows, 1 Dockerfile, 2 Compose files.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | How this plan complies | Status |
| --- | --- | --- |
| **I. Financial Correctness** | No money handling yet. Problem bodies never contain financial data (R7). The `Money` record arrives with the first spec that needs it. | ✅ N/A for now |
| **II. Non-Blocking I/O** | WebFlux/Netty, R2DBC, JDBC for Flyway only at startup. Security WebClient for JWKS has timeouts. AWS config imports run once at startup, not on a request path (R11). BlockHound in ITs, or the decided fallback (R15). Request data in Reactor `Context` and baggage, no `ThreadLocal` (R8). | ✅ (spike R15) |
| **III. Dates and Time** | Injected `Clock` bean; JWT clock-skew validation and `created_at`/`updated_at` use it. `timestamptz` UTC conventions recorded (data-model §2). | ✅ |
| **IV. Contract-First API** | OpenAPI 3.1 in `src/main/resources/openapi/`, `spring`/`reactive`/`interfaceOnly` generator (R3). Published per merge (R6). oasdiff breaking check, blocking from contract `1.0.0` (R5, constitution v1.1.0). Request/response validation in ITs (R4). `/api/v1`, kebab-case. RFC 9457 with `code`, `errors`, `requestId` (R7). Status codes per §IV. Unknown JSON fields rejected. `Cache-Control: no-store` on every response (R8). | ✅ |
| **V. Persistence** | R2DBC + `r2dbc-pool`; Flyway over a separate JDBC connection as the owner role (R13). Every database-touching service method is `@Transactional` (ArchUnit-enforced). Shared version-check and create-with-ID helpers in `persistence`. UUID PKs with client-supplied create semantics. `version` with `WHERE version = ?` → 409 `STALE_VERSION`. Forward-only migrations. PostgreSQL 17 on RDS and Aurora (R1). | ✅ |
| **VI. Security and Privacy** | Two caller kinds on separate chains; service tokens rejected on `/api/**` and user tokens on `/internal/**` (R9). Separate internal contract, not published. JWT validation of issuer, `client_id`, expiry, `token_use`, scopes. Owner-scoped repositories enforced by ArchUnit; 404 for foreign records. RLS-ready roles and `user_id` on every table. Log privacy rules plus test. `@Validated` config; refuses `localhost` and a non-Cognito issuer outside `local`/`test`. No auth bypass. dependency-check plus Trivy plus Dependabot, with dated suppressions. | ✅ |
| **VII. Code Organization** | `api`, `service`, `domain`, `persistence`, `config`, `common`, plus generated packages (R2). ArchUnit layering rules. DTOs stay out of the domain. Records, no Lombok/MapStruct. `@NullMarked` packages with NullAway. | ✅ |
| **VIII. Budget Domain Rules** | Not touched. Sort-key and `user_id` conventions recorded for later tables. | ✅ N/A for now |
| **IX. Test-First** | Platform behaviors are contract behaviors, so tests are written first (tasks will order them). JUnit 6, AssertJ, StepVerifier, WebTestClient, Testcontainers, BlockHound; no H2 or mocked DB. JaCoCo 85% on `domain` configured. ITs cover a write with returned state, stale version, cross-user 404 (test-only record). Compose stack with the mock OIDC and the real OIDC flow. Sample data only in the dev seed (guarded). | ✅ |
| **X. Performance** | No queries beyond test-only ones. Owner-leading index convention recorded. Every outbound call (JWKS, AWS config) has a timeout. Netty and pool use defaults. | ✅ |
| **XI. Observability** | ECS JSON logs to stdout; request ID and trace ID on every line via baggage and context propagation; user ID slot reserved. Micrometer metrics with a tag allow-list. OTel traces. Management port 8081 with `health` and `info` only. Privacy test. | ✅ |
| **Scope / Tech constraints** | Java 25, Boot 4.1, Maven Wrapper, `spring-boot-starter-parent`. Docker image. Environments `local`, `test`, `production`. | ✅ |
| **Workflow and gates** | `./mvnw verify` enforces tests, JaCoCo, ArchUnit, Spotless, Error Prone/NullAway, OpenAPI lint/breaking, dependency scan (R19). Pre-commit hook. Branch `001-platform-skeleton`. Conventional commits with `(001)` and task IDs. | ✅ |

**Gate result (pre-research)**: PASS. No violations. One compatibility spike (R15) has a decided
fallback. The R4 validator and the R11 Spring Cloud AWS imports are each proven by a first task.
Only the R15 fallback would be a deviation (BlockHound is named in API §II), and it would be
recorded below if triggered.

**Gate result (post-design)**: PASS. The design artifacts introduce nothing beyond the table
above. The test-only endpoint and table live entirely in `src/test` (R14), so no sample data
reaches production (API §IX).

## Project Structure

### Documentation (this feature)

```text
specs/001-platform-skeleton/
├── plan.md              # This file
├── research.md          # Phase 0: decisions R1–R21
├── data-model.md        # Phase 1: roles, conventions, test-only record, config, context
├── quickstart.md        # Phase 1: run and validation guide
├── contracts/
│   ├── budgit-api.yaml           # Public OpenAPI 3.1 contract (shared components, no paths)
│   ├── budgit-internal-api.yaml  # Internal OpenAPI 3.1 contract (not published)
│   └── problem-codes.md          # Problem code registry, headers, path kinds
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
pom.xml                                  # spring-boot-starter-parent 4.1.x; all gates bound to verify
mvnw, mvnw.cmd, .mvn/wrapper/            # Maven Wrapper
.mvn/jvm.config                          # JDK exports for Error Prone
.githooks/pre-commit                     # spotless:apply on staged Java files
redocly.yaml                             # contract lint rules
scripts/openapi-breaking-check.sh        # oasdiff vs origin/main baseline
Dockerfile                               # multi-stage, Temurin 25 JRE, non-root
compose.yaml                             # local: postgres, mock-oidc, api, ui (profile), seed (profile)
compose.ci.yaml                          # test overrides for CI E2E
docker/
├── postgres/init/01-roles.sql           # roles, database, budgit.environment setting
└── mock-oidc/config.json                # issuer "budgit", Cognito-like claims per client
dev/seed/000_guard.sql                   # aborts unless budgit.environment = 'local'
dependency-check-suppressions.xml        # suppressions with notes + until dates
.trivyignore.yaml                        # image-scan ignores with statement + expired_at
.github/
├── dependabot.yml
└── workflows/
    ├── ci.yml                           # verify, image (build + Trivy + prod-shaped smoke), e2e
    ├── publish-contract.yml             # release contract-v{major}.{minor}.{run}
    └── security-scan.yml                # scheduled dependency + image scan

src/main/java/com/danekoval/budgit/
├── BudgitApplication.java
├── api/                                 # controllers (none yet), ProblemDetailsAdvice, error WebExceptionHandler
│   └── filter/                          # RequestIdWebFilter, CacheControlWebFilter, ReadinessGateWebFilter,
│   │                                    # RequestLoggingWebFilter
├── service/                             # (empty; package-info)
├── domain/                              # (empty; package-info)
├── persistence/                         # VersionedUpdates, IdempotentCreates, OwnerScoped support
├── config/                              # BudgitProperties, SafeForEnvironment validator, SecurityConfig,
│                                        # ObservabilityConfig (MeterFilter allow-list), ClockConfig
└── common/                              # UserId, BudgetId, RequestId, ProblemCode, platform exceptions,
                                         # @UnscopedQuery
src/main/resources/
├── application.yaml, application-local.yaml, application-test.yaml, application-production.yaml
├── openapi/public/budgit-api.yaml        # ← contracts/budgit-api.yaml
├── openapi/internal/budgit-internal-api.yaml
└── db/migration/V1__baseline.sql         # schema budgit, grants, default privileges

src/test/java/com/danekoval/budgit/
├── architecture/ArchitectureTest.java    # layering, ownership params, @Transactional, banned APIs
├── support/                              # Testcontainers singletons, TokenFactory (mock OIDC),
│                                         # ContractValidatingFilter, LogCapture helpers
├── platform/                             # *IT: errors/status codes, request ID, no-store, auth chains,
│                                         # management port, readiness/503, DB roles, LogPrivacyIT
├── testwidget/                           # test-only controller/service/repository + OwnershipAndVersionIT
└── config/                               # BudgitProperties validation unit tests, AWS loader tests
src/test/resources/
├── openapi/test-widgets.yaml             # test-only contract ($refs public components)
├── db/test-schema.sql                    # owned_widget (test-only)
├── shared-fixtures/README.md             # published with the contract; empty for now
└── archunit.properties
```

**Structure Decision**: single Maven module at the repository root, with packages by layer per
API §VII. Generated OpenAPI code goes to `target/generated-sources/openapi` under
`com.danekoval.budgit.generated.{publicapi,internalapi}`. Test-only code proving PLT-4/PLT-5
lives entirely under `src/test`.

## Implementation Notes for `/speckit-tasks`

- **Order**: run spike R15, the R4 validator self-tests and the R11 import test first. Then go build skeleton → gates →
  error/headers → security → persistence helpers → observability → config → Docker/Compose →
  CI. Each phase writes its tests first (API §IX).
- **SC-004 (every gate blocks a violation)**: needs one throwaway verification per gate. Do these
  as a checklist in the PR, not as committed failing code.
- **Docker**: not installed on the current dev machine (checked 2026-10-12). Installing a Docker
  runtime is a prerequisite task.
- **First breaking-change check**: has no baseline. The first PR to `main` adds the contract, and
  the check activates from the second.

## Complexity Tracking

No constitution violations. If spike R15 fails and the BlockHound fallback is used, record it
here:

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| *(none)* | | |
