# Research: Platform Skeleton

**Feature**: 001-platform-skeleton | **Date**: 2026-10-12 | **Plan**: [plan.md](plan.md)

Each entry records a decision, why it was chosen, and what else was considered. Items marked
**Spike** are decisions whose compatibility with Java 25 / Spring Boot 4.1 could not be confirmed
from documentation; `tasks.md` should schedule each spike first, and the fallback is already
decided so a failed spike does not reopen the plan.

---

## R1. Versions and baseline

- **Decision**: Java 25; `spring-boot-starter-parent` 4.1.x (4.1.1 is the latest patch as of
  2026-08-20, so use the latest 4.1 patch at implementation time); PostgreSQL **17** in
  Docker, Testcontainers and (later) RDS.
- **Rationale**: Boot 4.1 is the latest GA line (released 2026-06-10) and builds on Spring
  Framework 7.0.x, as API Stack requires. PostgreSQL 17 is supported by both RDS for PostgreSQL
  and Aurora PostgreSQL. API §V requires a major version both support, and 18's Aurora
  availability is not something to depend on yet.
- **Alternatives**: Boot 4.0.x (older; the constitution asks for latest GA). PostgreSQL 18
  (newer, but Aurora parity is uncertain; can upgrade later through config only).

## R2. Base package and module layout

- **Decision**: single Maven module, base package `com.danekoval.budgit`, with the layer packages
  from API §VII: `api`, `service`, `domain`, `persistence`, `config`, `common`, plus
  `generated` (OpenAPI output, under `target/generated-sources`, package
  `com.danekoval.budgit.generated.{publicapi,internalapi}`).
- **Rationale**: API §VII names these layers; one module is the simplest structure that works.
  `groupId` stays `com.danekoval`, and `budgit` matches the product name.
- **Alternatives**: multi-module (allowed by §VII only once the codebase outgrows one module).

## R3. Contract-first code generation

- **Decision**: `openapi-generator-maven-plugin`, generator `spring`, `reactive=true`,
  `interfaceOnly=true`, `useJakartaEe=true`, `useBeanValidation=true`, Boot 4 / Spring 7 option
  enabled (use whichever `useSpringBoot*` flag the current generator version provides for Boot 4).
  It runs twice: once for `openapi/public/budgit-api.yaml` → `generated.publicapi`, and once
  for `openapi/internal/budgit-internal-api.yaml` → `generated.internalapi`. The Problem schema
  maps to Spring's `org.springframework.http.ProblemDetail` (`schemaMappings`/`importMappings`),
  so errors use one Java type.
- **Rationale**: API §IV prescribes exactly this generator and options. Two runs keep the
  internal contract out of the public artifact (FR-006).
- **Alternatives**: one spec with tags (rejected: the internal surface would leak into the
  published contract).
- **Note**: the skeleton's contracts have no business paths, so generation yields only the shared
  models. That is expected.

## R4. Contract validation in integration tests (FR-008)

- **Decision**: Atlassian `com.atlassian.oai:openapi-request-validator-core` **3.x**
  (`OpenApiInteractionValidator`), behind a small `ExchangeFilterFunction` registered on every
  `WebTestClient`. The filter validates each request/response pair and fails the test on any
  report message at ERROR level.
- **Why it fits**: per the project's release notes, v3.0.0:
  - adds full OpenAPI 3.1 validation, backed by JSON Schema 2020-12 (type unions, `const`,
    `if/then/else`, `prefixItems`, `unevaluatedProperties` and more);
  - targets Spring Framework 7 / Spring Boot 4 (Jakarta);
  - requires Java 21+;
  - renamed the artifacts from `swagger-request-validator-*` to `openapi-request-validator-*`
    (the group ID is unchanged).

  Beyond body schemas, it checks path matching, parameters, headers, content types and declared
  status codes.
- **Adapter**: no WebFlux/`WebTestClient` adapter module is listed, so the filter that converts
  `WebTestClient` exchanges into the core library's request/response objects is our own (small)
  code.
- **Validator self-tests (first task)**: known-good and known-bad request/response pairs against
  our 3.1 contract, including 3.1-specific constructs (e.g. a nullable field written as
  `type: [string, "null"]`, `const`). They prove the validator catches what it should and does not
  silently pass invalid responses.
- **Fallback (if the self-tests fail)**: the same filter, implemented with `swagger-parser`
  (operation and schema resolution, 3.1-aware) plus `networknt json-schema-validator` (JSON Schema
  2020-12), checking status code, content type, required headers and body.
- **Alternatives**: downgrading the contract to 3.0 (rejected: API §IV mandates 3.1).

## R5. Contract lint and breaking-change check (FR-010)

- **Decision**: both run inside `./mvnw verify` via `exec-maven-plugin`, using pinned Docker
  images (Docker is already required by Testcontainers):
  - **Lint**: Redocly CLI (`redocly lint`) with a committed `redocly.yaml` (recommended ruleset
    plus rules: `operation-operationId`, kebab-case paths, `/api/v1` prefix on public paths,
    every operation declares the shared error responses).
  - **Breaking changes**: `oasdiff breaking --fail-on ERR` between the PR's public contract and
    the baseline from `origin/main` (`git show origin/main:<path>`). A script in
    `scripts/openapi-breaking-check.sh` does this. Each skip prints a log line:
    - **No baseline** (first PR): skipped.
    - **`info.version` below `1.0.0`** (pre-release, constitution v1.1.0): oasdiff still runs
      and prints its report, but doesn't fail the build.
    - **`info.version`'s major number increased**: skipped.
- **Rationale**: API §IV names `oasdiff`. Redocly has the most complete 3.1 lint support. Running
  both through Docker avoids a Node toolchain in a Java build.
- **Alternatives**: Spectral (also good, less 3.1-specific). OpenAPITools `openapi-diff` Maven
  plugin (Java-native, but weaker 3.1 support). `frontend-maven-plugin` + npm (adds a Node
  toolchain).

## R6. Publishing the contract and shared fixtures (FR-009)

- **Decision**: a GitHub Actions workflow on push to `main` (after `verify` passes) bundles the
  public contract into one file (`redocly bundle`), sets its `info.version` to
  `{major}.{minor}.{run_number}` (major/minor from the committed `info.version`), and creates a
  GitHub release `contract-v{version}` with two assets: `budgit-api-{version}.yaml` and
  `shared-fixtures-{version}.tar.gz` (from `src/test/resources/shared-fixtures/`).
- **Rationale**: API §IV suggests a GitHub release asset. The version is unique per merge,
  ordered, and still tells the UI which major/minor it pins. Humans bump major/minor in the
  source contract.
- **Alternatives**: GitHub Packages Maven artifact (awkward for a TypeScript consumer); npm
  package (needs a registry and token, can come later); semver build metadata `+run` (not
  ordered by tools).

## R7. Problem Details and status codes (FR-011, FR-012)

- **Decision**: Spring's `ProblemDetail` everywhere, produced by:
  - a `@RestControllerAdvice` extending WebFlux `ResponseEntityExceptionHandler` (validation,
    415, 413, 405, 406, 400 for unreadable/unknown JSON fields, domain exceptions);
  - a `WebExceptionHandler` (order before Boot's default) for errors outside controllers, such as
    an unknown path (404);
  - Spring Security's `ServerAuthenticationEntryPoint` for 401.

  Each body carries `type`, `title`, `status`, `code`, `requestId`, and optional `errors`
  (`[{field, code}]`). `detail` is never populated from exception messages. Stable codes are
  defined in [contracts/problem-codes.md](contracts/problem-codes.md).
- **Codes and leakage**: every token failure → `401 UNAUTHENTICATED`, and every missing or
  foreign record → `404 NOT_FOUND`, with identical bodies apart from `requestId` (spec edge case
  on 401/404). The cause is logged at DEBUG without token values.
- **Unknown JSON fields**: `spring.jackson.deserialization.fail-on-unknown-properties=true`
  (set explicitly; do not rely on the Jackson 3 / Boot default) → `400 VALIDATION_FAILED`.
- **413**: `spring.http.codecs.max-in-memory-size` (default 256 KB kept) →
  `DataBufferLimitException` → `413 PAYLOAD_TOO_LARGE`.
- **503**: R2DBC resource failures (`DataAccessResourceFailureException`, connection-pool
  acquire timeout) → `503 SERVICE_UNAVAILABLE`, with no `Retry-After` unless known. A `WebFilter`
  returns 503 while `ReadinessState` is `REFUSING_TRAFFIC` (during shutdown).
- **502/504**: a shared `ExternalCallException` hierarchy (`UpstreamFailed` → 502,
  `UpstreamTimeout` → 504) for future outbound calls. JWKS fetch failures during authentication
  → 503 (fail closed, never 200).
- **405/406**: not in API §IV's list but produced by the framework. They are kept as Problem
  Details (`METHOD_NOT_ALLOWED`, `NOT_ACCEPTABLE`), not remapped.
- **Alternatives**: hand-rolled error DTO from the contract (rejected: Spring's type already
  implements RFC 9457, and mapping it in the generator keeps one type).

## R8. Request ID, Cache-Control and request context (FR-013, FR-014, FR-023)

- **Decision**:
  - `RequestIdWebFilter` (highest precedence): accept `X-Request-Id` matching
    `^[A-Za-z0-9._-]{1,64}$`, otherwise generate a UUID. Put it in the Reactor `Context` and an
    exchange attribute, and add it to the response header via `beforeCommit`.
  - Logging correlation: Micrometer Tracing baggage field `requestId`
    (`management.tracing.baggage.correlation.fields=requestId`, not propagated remotely), so it
    lands in the MDC alongside `traceId`/`spanId`. Boot's
    `spring.reactor.context-propagation=auto` handles Reactor ↔ thread-local propagation. The
    internal user ID is added the same way by the next spec.
  - `CacheControlWebFilter` sets exactly `Cache-Control: no-store` on every response. Spring
    Security's default cache headers are disabled so they do not add `no-cache, max-age=0, ...`.
- **Rationale**: API §II forbids `ThreadLocal` for request data; baggage plus automatic
  propagation is the supported Spring way to get Context values into logs. Validating the
  incoming header prevents log injection (spec edge case).
- **Alternatives**: manual MDC handling per operator (error-prone); trusting the header verbatim
  (log injection).

## R9. Authentication: resource server, two caller kinds (FR-019, FR-020)

- **Decision**: `spring-boot-starter-oauth2-resource-server` with a `NimbusReactiveJwtDecoder`
  built from config:
  - **Trusted issuer** (`budgit.security.issuer-uri`) validates `iss`, and the JWKS URI is
    discovered from it or set explicitly. Nimbus caches the key set and refetches on an unknown
    `kid` (key rotation edge case). The JWKS `WebClient` has connect/read timeouts (2 s / 3 s).
  - **Validators**: issuer, `exp`/`nbf` with 60 s clock skew via the injected `Clock`,
    `token_use == "access"`, and `client_id` in the allowed set for the chain.
  - **Two `SecurityWebFilterChain`s**:
    - `/api/**`: `client_id` ∈ `budgit.security.user-client-ids`.
    - `/internal/**`: `client_id` ∈ `budgit.security.service-client-ids` **and** the
      endpoint's required scope (`SCOPE_budgit/<task>`).

    A service token on `/api/**` fails the client check → 401, and a user token on
    `/internal/**` fails too → 401. Anything else → 404 via the Problem handler (authentication
    happens first, so unknown paths under `/api/**` still require a token).
  - No profile disables security. Tests get real tokens from the mock OIDC server (R12).
- **Rationale**: Cognito access tokens have no `aud`. `client_id` is the documented way to bind
  them to an app client, and it cleanly separates user-facing from service callers (API §VI).
  Fail-closed, timeouts and no bypass are all API §VI requirements.
- **Alternatives**: separating callers by the presence of a `username` claim (fragile across
  IdPs); one chain with per-path rules (harder to reason about and test).

## R10. Startup configuration validation (FR-021, FR-022)

- **Decision**:
  - `budgit.*` properties are bound to `@Validated @ConfigurationProperties` records (Jakarta
    constraints on each field).
  - `budgit.environment` (enum `LOCAL`, `TEST`, `PRODUCTION`) is required and must equal the
    active Spring profile.
  - A class-level `ConstraintValidator` (`@SafeForEnvironment`) applies the cross-field rules.
    Outside `LOCAL`/`TEST` it rejects any configured URL or host that resolves to
    `localhost`/`127.0.0.0/8`/`::1`, unless the property is listed in
    `budgit.sidecar-endpoints` with a non-blank reason (a sidecar, API §VI v1.1.0). It also
    rejects a trusted issuer not matching
    `^https://cognito-idp\.[a-z0-9-]+\.amazonaws\.com/[A-Za-z0-9-]+_[A-Za-z0-9]+$`.
  - The failure message names the property, never its value.
  - Boot's failure analyzer stops startup.
- **Rationale**: this is exactly API §VI (as amended in v1.1.0) and keeps validation declarative.
  Declaring sidecars by property name, in committed config, keeps every allowed `localhost` visible
  in review.
- **Alternatives**: an `ApplicationRunner` check (runs too late; the server may already be
  listening).

## R11. Remote config (AppConfig) and secrets (Secrets Manager)

- **Decision**: Spring Cloud AWS **4.2.x** (`io.awspring.cloud:spring-cloud-aws-dependencies`
  BOM) for both sources, imported through `spring.config.import` in
  `application-production.yaml` only, with `optional:` so `local` and `test` need no AWS:
  - **Secrets**: `spring-cloud-aws-starter-secrets-manager` (`aws-secretsmanager:` import).
  - **AppConfig**: `spring-cloud-aws-starter-app-config` (AppConfig integration added in 4.1.0;
    4.2.0 fixes token refresh). It is imported after the committed file, so AppConfig values
    override it.
- **Evidence** (awspring releases and README, checked 2026-10-18):
  - The 4.x line is built for Spring Boot 4 and Spring Framework 7; 4.2.0 is the latest
    release (Sep 2026).
  - The README's compatibility table still lists only Boot 4.0.x, but 4.1.0 (Jul 2026, after
    Boot 4.1's release) and later "lift Spring versions to latest".
  - Secrets Manager and AppConfig are both listed as supported in 4.1.x.
- **First task** (replaces the earlier spike): confirm the exact import prefixes in the 4.2
  reference docs, and add an integration test that starts the app on Boot 4.1 with both imports
  pointed at a WireMock endpoint override. A failure there means pinning a newer Spring Cloud
  AWS patch, or as a last resort writing our own loaders on the AWS SDK v2 (`appconfigdata`,
  `secretsmanager`).
- **Upgrade risk**: Spring Cloud AWS releases a little after each Spring Boot minor. If it lags
  long enough to block the constitution's "upgrade Boot minors within a month", the same
  last-resort loaders apply.
- **Blocking note**: config loading happens once, at startup, before the server accepts traffic.
  It is not a request path, so API §II's non-blocking rule does not apply. Each call still has a
  timeout (5 s).
- **Not run anywhere yet**: production wiring is exercised only by tests and a production-shaped
  container start (spec Story 7 scenario 7). Real AWS comes with the deferred infrastructure spec.
- **Alternatives**:
  - The AppConfig Agent sidecar (`localhost:2772`): it adds a container to deploy and run, and
    Spring Cloud AWS already reads AppConfig directly. Sidecars aren't ruled out for later needs.
  - Our own loaders on the AWS SDK v2: more code to maintain, now that a supported library
    covers both sources. They remain the last resort.
  - Environment variables only: they don't meet "overridden by AWS AppConfig".

## R12. Mock OIDC provider and the issuer hostname

- **Decision**: `ghcr.io/navikt/mock-oauth2-server` (pinned). It supports authorization-code with
  interactive login (UI sign-in) and `client_credentials` (service callers). A committed
  `docker/mock-oidc/config.json` adds Cognito-like claims (`token_use=access`, `client_id`,
  `scope`, `username`).
  - **Canonical issuer**: `http://mock-oidc:8090/budgit`, using the same port inside and outside
    Docker, so every party sees the same `iss`. Developers add `127.0.0.1 mock-oidc` to
    `/etc/hosts` once for browser sign-in. The quickstart documents this, and the CI runner
    adds it the same way.
  - **Testcontainers**: the same image runs as a `GenericContainer` in integration tests. Tests
    request real tokens (user via a non-interactive token request, service via
    `client_credentials`).
- **Rationale**: it is lightweight and covers both flows required by PLT-8. The issuer must be
  identical for the browser, Next.js server and API, or validation fails; one hostname is the
  simplest guarantee.
- **Alternatives**: Keycloak (heavy, slower CI); Dex (no `client_credentials` convenience);
  `network_mode: host` (unsupported or unreliable on macOS Docker).

## R13. Persistence: roles, transactions, ownership, versions (FR-015 to FR-018)

- **Decision**:
  - **Roles**: `budgit_owner` (owns the database and schema `budgit`; used only by Flyway over
    JDBC) and `budgit_app` (LOGIN, NOSUPERUSER, no CREATE; used by R2DBC). Roles are created
    outside Flyway (cluster-level): by `docker/postgres/init/01-roles.sql` in Compose and
    Testcontainers, and by the deferred infrastructure spec in AWS.
  - **Migration V1**: creates schema `budgit`, grants `USAGE` to the app role (name via Flyway
    placeholder `${appRole}`), and sets
    `ALTER DEFAULT PRIVILEGES FOR ROLE budgit_owner IN SCHEMA budgit GRANT SELECT, INSERT, UPDATE,
    DELETE ON TABLES TO ${appRole}` (plus sequences), so every future table is usable but not
    alterable by the app.
  - **Flyway** runs via Boot's auto-configuration (`spring.flyway.url/user/password` = owner)
    before the web server starts, so a migration failure means no traffic. `spring.r2dbc.*` =
    app role. An R2DBC `ConnectionFactoryInitializer` is not used.
  - **Startup retry**: Flyway `connectRetries=10`, `connectRetriesInterval=3` (bounded wait for
    the database, spec edge case).
  - **Transactions**: services annotate public methods `@Transactional` (reactive transaction
    manager). An ArchUnit rule requires it on every public method of a `@Service` that depends
    on `persistence`.
  - **Ownership**:
    - Repositories extend `org.springframework.data.repository.Repository` (no inherited
      unscoped `findById` and similar) or use `DatabaseClient`.
    - An ArchUnit rule requires every public method of a class or interface in `persistence`
      to take a `UserId` or `BudgetId` parameter (typed ID records in `common`).
    - The only escape hatch is `@UnscopedQuery(reason = "...")`, needed by the next spec's
      lookup by Cognito `sub`.
  - **Version check**: a shared `VersionedUpdates` helper in `persistence` runs
    `UPDATE … SET …, version = version + 1, updated_at = :now WHERE id = :id AND user_id = :owner
    AND version = :expected RETURNING *`. On 0 rows it runs a scoped existence check:
    `SELECT 1 … WHERE id = :id AND user_id = :owner`. If the record exists →
    `StaleVersionException` (409 `STALE_VERSION`); if not → `NotFoundException` (404).
  - **Create with client ID**: `INSERT … ON CONFLICT (id) DO NOTHING RETURNING *`. On no row, a
    scoped select by `(id, owner)` returns the existing record if found; otherwise →
    `IdConflictException` (409 `ID_CONFLICT`).
- **Rationale**: these choices meet API §V/§VI exactly. Ownership is enforced structurally (the
  type system plus ArchUnit), not by convention.
- **Alternatives**: Spring Data `ReactiveCrudRepository` (exposes unscoped methods); RLS policies
  now (deferred by MVP §3); jOOQ (allowed later; not needed for the skeleton).

## R14. Proving PLT-4/PLT-5 without production tables

- **Decision**: a test-only `owned_widget` table and a test-only endpoint `/api/v1/test-widgets`.
  - **Table**: created by `src/test/resources/db/test-schema.sql`, run as the owner role after
    Flyway in the integration-test setup. It is not a Flyway migration, so it never enters
    migration history or production.
  - **Endpoint**: lives in `src/test/java`, with a matching test-only contract
    `src/test/resources/openapi/test-widgets.yaml` that `$ref`s the public components. The
    contract validator loads it for those paths.
- **Rationale**: the spec assumption requires proving the mechanisms with test-only records. Using
  real production code (the helpers, exception handling, security chain) through a test endpoint
  exercises everything except the table itself.
- **Alternatives**: a Flyway migration in a test-only location (pollutes version ordering); only
  unit-testing the helpers (would not prove 404/409 end to end).

## R15. Non-blocking enforcement (FR-004) — **Spike**

- **Decision**: BlockHound with `blockhound-junit-platform` (auto-installs for the test JVM),
  with `-XX:+AllowRedefinitionToAddDeleteMethods` in the Failsafe `argLine` for integration
  tests.
- **Spike**: there is no confirmed JDK 25 support. The flag still exists in JDK 25 (it is slated
  for obsoletion in JDK 26), and there is a user report of startup failure on JDK 22. Spike it
  first.
- **Fallback (decided)**: if BlockHound cannot run on JDK 25:
  - an Error Prone custom check / ArchUnit rule banning `.block*()`, `toFuture().get()`,
    `Thread.sleep`, `java.io` and `java.net` blocking APIs in `src/main` outside
    `Mono.fromCallable(...).subscribeOn(boundedElastic())` call sites (static);
  - an integration-test assertion that request handling runs only on `reactor-http-nio-*`
    threads without calls to the banned APIs (dynamic, via a `Schedulers.onScheduleHook`
    sanity check).

  This deviation would be recorded in Complexity Tracking, as API §II names BlockHound
  explicitly.
- **Alternatives**: running integration tests on an older JDK (rejected: diverges from
  production).

## R16. Logging, metrics, tracing and the privacy test (FR-023 to FR-026)

- **Decision**:
  - **Logs**: Boot structured logging `logging.structured.format.console=ecs` to stdout only; no
    file appenders. Request/response body logging stays off (no `logging.level` for codecs or
    wiretap). The startup banner is off (`spring.main.banner-mode=off`), since it isn't JSON.
  - **Request completion log**: a `RequestLoggingWebFilter` writes one INFO line per request when
    the response completes. Fields: `http.method`, `url.template` (the route pattern, never the
    actual URL or query string), `http.status`, `duration.ms`, plus the usual `requestId` and
    `trace.id`. It never logs headers or bodies.
    - **Why our own filter:** it runs inside WebFlux's processing, while the request's context
      is active, so the line carries `requestId` and `trace.id` as JSON fields.
    - **Not Reactor Netty's built-in access log:** that log is written by Netty after WebFlux
      has finished, so its lines have no `requestId` or `trace.id`, and its fields end up as a
      string inside `message`.
    - **Also used by checks:** every request then produces at least one line for the quickstart
      check and the privacy test to inspect.
  - **Tracing**: Micrometer Tracing with the OpenTelemetry bridge and OTLP exporter (Boot 4.1
    OpenTelemetry starter). Export is disabled in `local`/`test` (traces still generated, so IDs
    exist) and points at an OTLP endpoint in `production` (the ADOT collector arrives with the
    infrastructure spec).
  - **Metrics**: Micrometer with the OTLP registry, on the same terms. `http.server.requests`
    uses URI templates only. A `MeterFilter` denies any tag key outside an allow-list
    (`method`, `uri`, `status`, `outcome`, `exception`, `error`), so a user ID or value can never
    become a metric tag.
  - **Management**: `management.server.port=8081`, exposure `health,info` only, probes enabled.
    Readiness includes `readinessState` and the R2DBC indicator; liveness is `livenessState`
    only. Info includes build info (Maven `build-info` goal) and git commit.
  - **Privacy test** (`LogPrivacyIT`):
    - **Inputs**: requests carrying sentinel values in headers, query and body, such as an
      amount like `987654321`, `SENTINEL-PAYEE-…`, an email, and the bearer token itself.
    - **Captured**: stdout (`OutputCaptureExtension`), finished spans (an in-memory OTel span
      exporter bean in tests), and meter tags (`SimpleMeterRegistry`).
    - **Asserts**: no sentinel appears anywhere; every captured line parses as JSON; and every
      line logged while handling the test's requests (matched by the request IDs it sent) has
      `requestId` and `trace.id`.
- **Rationale**: API §XI. ECS is a widely understood schema that CloudWatch Logs Insights handles
  well.
- **Alternatives**: Logstash JSON format (equivalent; ECS chosen for field naming); a vendor agent
  (ships data itself, which API §XI forbids).

## R17. Container image (FR-027)

- **Decision**: a multi-stage `Dockerfile`:
  - **Build stage**: Temurin 25 JDK, `./mvnw -DskipTests package`.
  - **Runtime stage**: `eclipse-temurin:25-jre` pinned by digest; the jar extracted with
    `java -Djarmode=tools extract --layers --launcher` for layer caching; a non-root user;
    ports 8080 (app) and 8081 (management); `HEALTHCHECK` none (the orchestrator uses
    readiness).
  - **JVM warnings:** plain-text warnings the JVM prints (e.g. about restricted native access or
    deprecated options) are **not** suppressed. They signal things that may break on a future
    Java upgrade.
    - **Flags:** one that changes JVM behaviour (e.g. `--enable-native-access`) is added only as
      a deliberate decision, with a comment saying why. It is never added to tidy logs.
    - **Visibility:** the CI `image` job's production-shaped start prints any non-JSON startup
      lines as a report, without failing the build, so a new warning is noticed.
- **Not in the image**: `docker/`, `dev/seed/` and test resources (the dev seed cannot ship;
  FR-029).
- **Rationale**: full control over base image, user and contents, and easy to scan.
- **Alternatives**: Boot buildpacks `build-image` (less control over contents; still fine later);
  a distroless Java 25 base (availability unverified; can switch later).

## R18. Compose stack and dev seed (FR-028, FR-029)

- **Decision**: `compose.yaml` (local) plus a `compose.ci.yaml` override (test). The ports are
  example values.

  | Service | Image | Host port | Notes |
  | --- | --- | --- | --- |
  | `postgres` | `postgres:17` | 5432 | Mounts `docker/postgres/init/` (roles, database, and `ALTER DATABASE budgit SET budgit.environment = 'local'`; `test` in the CI override). |
  | `mock-oidc` | mock-oauth2-server | 8090 | Same port inside and outside Docker (R12). |
  | `api` | built from the `Dockerfile` | 8080 app, 8081 management | Depends on `postgres` healthy and `mock-oidc` started; `SPRING_PROFILES_ACTIVE=local` (`test` in CI). |
  | `ui` | latest released `budget-app` image | 3000 | Behind Compose profile `ui`, off until the UI publishes a release (spec assumption). |
  | `seed` | `postgres:17` running `psql` | n/a | Profile `seed`, local file only, runs `dev/seed/*.sql` as the app role. |

  `dev/seed/000_guard.sql` aborts unless `current_setting('budgit.environment', true) = 'local'`.
  It is otherwise empty; later specs add their sample data files.
- **Rationale**: one stack for both uses (API §IX). The guard is enforced by the database itself,
  so the seed cannot run against a `test` or production database even if pointed at one by
  mistake.
- **Alternatives**: seed as a Flyway location enabled only in `local` (risks migration-history
  drift between environments); Spring `CommandLineRunner` seed (would ship in the image).

## R19. Quality gates in `./mvnw verify` (FR-030)

- **Decision**: all of these bind to `verify` (or earlier phases) in `pom.xml`:
  - **Surefire**: unit tests (`*Test`). **Failsafe**: integration tests (`*IT`, Testcontainers).
  - **JaCoCo**: `check` with LINE ≥ 0.85 for `com.danekoval.budgit.domain.**`. An empty package
    passes (no counters); it becomes meaningful with the first domain logic.
  - **ArchUnit**: rules in a JUnit test (`ArchitectureTest`); `archunit.properties` sets
    `archRule.failOnEmptyShould=false` while layers are empty.
  - **Spotless**: Palantir Java Format `check`. A committed `.githooks/pre-commit` runs
    `spotless:apply` on staged Java files, enabled once via `git config core.hooksPath .githooks`
    (quickstart).
  - **Error Prone + NullAway**:
    - Wired through `maven-compiler-plugin` `annotationProcessorPaths`; JDK module exports go in
      `.mvn/jvm.config`.
    - NullAway uses `OnlyNullMarked=true` with JSpecify `@NullMarked` in every `package-info.java`.
    - Generated sources are excluded (`-XepExcludedPaths:.*/generated-sources/.*`).
  - **OpenAPI lint + breaking-change check**: R5.
  - **Dependency scan**: OWASP `dependency-check-maven`:
    - `failBuildOnCVSS=7`, with a suppression file whose entries require `<notes>` (the why) and
      an `until` date (re-check), matching API §VI.
    - NVD API key from `NVD_API_KEY`; the NVD data cache is shared across CI runs.
    - Locally, `-Ddependency-check.skip=true` is allowed for speed (documented). CI never skips.
- **Rationale**: these are API Workflow's gates verbatim, all reproducible locally with one
  command.
- **Alternatives**:
  - Trivy filesystem scan for dependencies (good, but lives outside Maven; used for the image
    instead).
  - OSS Index (now requires credentials; dependency-check is the de facto Maven choice).

## R20. GitHub Actions (FR-030, FR-031)

- **Decision**: three workflows:
  - **`ci.yml`** (on `pull_request` and push to `main`):
    - `verify` job: Temurin 25, Maven cache, NVD cache, `./mvnw -B verify`.
    - `image` job: build the image, then Trivy image scan failing on HIGH/CRITICAL, with ignores
      in `.trivyignore.yaml` using `expired_at` dates plus a `statement`.
    - `e2e` job: `docker compose -f compose.yaml -f compose.ci.yaml up --wait` with the PR's
      image, then an API smoke suite (health, 401 Problem Details, request ID, `no-store`,
      tokens from the mock server). The UI's E2E suite is added to this job once `budget-app`
      publishes a runnable suite and release.
  - **`publish-contract.yml`** (push to `main`, needs `ci` success): R6.
  - **`security-scan.yml`** (`schedule`: daily; plus `workflow_dispatch`): dependency-check plus
    a Trivy scan of an image built from `main`, failing visibly.
  - **Also**: `.github/dependabot.yml` for `maven`, `github-actions` and `docker` ecosystems
    (weekly).
- **Branch protection**: manual repository settings (`main` requires the `verify`, `image` and
  `e2e` checks and a pull request). The quickstart documents them.
- **Rationale**: API Workflow and §VI. Splitting image and E2E into their own jobs keeps
  `verify` identical to the local command.
- **Alternatives**: one monolithic job (slower feedback; mixes concerns).

## R21. Environment profiles (FR-032, FR-033)

- **Decision**: Spring profiles `local`, `test` and `production`, each with an
  `application-{profile}.yaml` alongside a shared `application.yaml`.
  - **Integration tests**: activate `test` via `@ActiveProfiles("test")` and inject container
    URLs with `@DynamicPropertySource` / `@ServiceConnection`.
  - **Production**: `application-production.yaml` holds the non-secret settings that are already
    known. It doesn't contain connection hosts (database, issuer, telemetry) yet, because the
    resources don't exist until the deferred infrastructure spec creates them.
    - **Infrastructure spec decides:** whether hosts are then committed to this file or injected
      from the infrastructure code's outputs (e.g. environment variables in the task definition).
      The trade-off: committed values are visible and reviewed in Git; injected values keep the
      infrastructure code as the single source of truth.
    - **Either way:** secrets are never committed and come only from Secrets Manager or
      environment variables (API §VI). A missing value fails startup validation (R10).
  - **Production-shaped smoke test** (CI `image` job, not `verify`): the built image runs with
    the `production` profile on a Docker network next to Postgres (alias `db`) and the mock OIDC
    server (alias `mock-oidc`). It is configured with:
    - a Cognito-pattern issuer string (`https://cognito-idp.us-east-1.amazonaws.com/us-east-1_Smoke1`);
    - an explicit JWKS URI pointing at the mock (keys are fetched lazily, so no AWS call happens
      at startup);
    - secrets as environment variables, with the AppConfig and Secrets Manager imports left
      unset (both are `optional:`).

    It asserts the service reaches ready, and then that a missing variable or a `localhost`
    host makes it exit (spec Story 7 scenarios 1–3 and 7).
- **Rationale**: matches API Environments and the spec's option-C scope.
- **Alternatives**: a fourth `ci` profile (unnecessary; CI is `test`).
