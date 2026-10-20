---

description: "Task list for feature 001-platform-skeleton"
---

# Tasks: Platform Skeleton

**Input**: Design documents from `specs/001-platform-skeleton/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Included. API constitution §IX requires test-first development for each endpoint's
contract behavior, and these platform behaviors are contract behavior. Within each story, write the
tests first and confirm they fail before implementing.

**Organization**: Tasks are grouped by user story (spec.md) so each story can be implemented and
verified on its own, after the Setup and Foundational phases.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependency on an incomplete task)
- **[Story]**: The user story the task belongs to (US1–US7)
- Every task names the file(s) it creates or changes

## Path Conventions

- Production code: `src/main/java/com/danekoval/budgit/` (written below as `main/…`)
- Production resources: `src/main/resources/`
- Test code: `src/test/java/com/danekoval/budgit/` (written below as `test/…`)
- Test resources: `src/test/resources/`
- Everything else is at the repository root.

`main/…` and `test/…` are shorthand for those two Java source roots only; all other paths are
literal. Package names follow the path (e.g. `main/api/filter/` → `com.danekoval.budgit.api.filter`).

## Conventions every task follows

- Constitution v1.1.0 applies throughout (`.specify/memory/constitution.md`). Key rules for this
  feature:
  - no blocking calls on request paths;
  - an injected `Clock` for all time;
  - records for data classes, with no Lombok or MapStruct;
  - every package has `package-info.java` annotated `@NullMarked` (JSpecify);
  - no financial values, tokens or emails in logs, metrics or traces;
  - `Optional` is used only as a return type.
- Commits follow `feat(001): <summary> [Txxx]` (Conventional Commits with the task ID).
- **Versions**: use the latest patch of each line named here, and check it at implementation time:
  - Spring Boot 4.1.x;
  - Spring Cloud AWS 4.2.x;
  - `com.atlassian.oai:openapi-request-validator-core` 3.x;
  - Testcontainers, ArchUnit, Error Prone, NullAway, Spotless, JaCoCo, OWASP dependency-check:
    latest stable.
- Container images are pinned by version tag in Compose and by digest in the Dockerfile:
  `postgres:17`, `ghcr.io/navikt/mock-oauth2-server`, `redocly/cli`, `tufin/oasdiff`,
  `aquasec/trivy`, `eclipse-temurin:25-jdk` and `eclipse-temurin:25-jre`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: a compiling, formatted, statically checked empty Spring Boot project, plus the one
remaining compatibility spike.

- [ ] T001 Add the Maven Wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/maven-wrapper.properties`) pinned to the latest Maven 3.9.x by running `mvn -N wrapper:wrapper` once, then commit the files. `./mvnw -v` must work with no global Maven installed (FR-001).
- [ ] T002 Rewrite `pom.xml`:
  - **Coordinates and parent**: keep `groupId` `com.danekoval` and `artifactId` `budget-api`; set `version` to `0.1.0-SNAPSHOT`; set parent `spring-boot-starter-parent` 4.1.x; set `<java.version>25</java.version>`.
  - **BOM**: import `io.awspring.cloud:spring-cloud-aws-dependencies` 4.2.x in `dependencyManagement`.
  - **Main dependencies**:
    - Spring Boot starters: `spring-boot-starter-webflux`, `spring-boot-starter-security`, `spring-boot-starter-oauth2-resource-server`, `spring-boot-starter-data-r2dbc`, `spring-boot-starter-validation`, `spring-boot-starter-actuator`, `spring-boot-starter-flyway` and `spring-boot-starter-opentelemetry` (or the Boot 4.1 equivalents for Flyway and OpenTelemetry).
    - Database: `org.postgresql:r2dbc-postgresql` and `io.r2dbc:r2dbc-pool`; `org.flywaydb:flyway-database-postgresql`; `org.postgresql:postgresql` (runtime, migrations only).
    - Telemetry: `io.micrometer:micrometer-registry-otlp`.
    - Null-safety: `org.jspecify:jspecify`.
    - AWS: `io.awspring.cloud:spring-cloud-aws-starter-secrets-manager` and `io.awspring.cloud:spring-cloud-aws-starter-app-config`.
  - **Test dependencies**:
    - `spring-boot-starter-test`, `spring-boot-testcontainers`, `io.projectreactor:reactor-test`.
    - Testcontainers: `org.testcontainers:testcontainers`, `org.testcontainers:postgresql` and `org.testcontainers:junit-jupiter`.
    - `com.tngtech.archunit:archunit-junit5`.
    - BlockHound: `io.projectreactor.tools:blockhound` and `io.projectreactor.tools:blockhound-junit-platform`.
    - `com.atlassian.oai:openapi-request-validator-core` 3.x.
    - `org.wiremock:wiremock-standalone`.
  - **Plugins** (configured by later tasks, declared here):
    - `spring-boot-maven-plugin` with the `build-info` goal;
    - `maven-surefire-plugin` (includes `**/*Test.java`);
    - `maven-failsafe-plugin` (includes `**/*IT.java`; `integration-test` and `verify` goals).
- [ ] T003 [P] Create `main/BudgitApplication.java` (`@SpringBootApplication`, `main` method) and a `package-info.java` with `@NullMarked` in each of `main/`, `main/api/`, `main/api/filter/`, `main/service/`, `main/domain/`, `main/persistence/`, `main/config/` and `main/common/`. Also create empty `src/main/resources/application.yaml` with `spring.application.name: budget-api`.
- [ ] T004 [P] Create `.gitignore` additions (`target/`, `.idea/` already present: keep), `.dockerignore` (exclude `target/`, `.git/`, `.idea/`, `docker/`, `dev/`, `specs/`, `docs/`, `src/test/`), and `.editorconfig` (UTF-8, LF, 4-space Java indent, 2-space YAML).
- [ ] T005 Configure Spotless in `pom.xml`:
  - `spotless-maven-plugin` with Palantir Java Format for `src/**/*.java`;
  - goal `check` bound to the `validate` phase;
  - exclude `target/generated-sources/**`.

  Run `./mvnw spotless:apply` once.
- [ ] T006 Configure Error Prone and NullAway in `pom.xml`:
  - **Compiler setup**: `maven-compiler-plugin` with `-XDcompilePolicy=simple`, `--should-stop=ifError=FLOW`, and `-Xplugin:ErrorProne -XepDisableWarningsInGeneratedCode -XepExcludedPaths:.*/generated-sources/.* -Xep:NullAway:ERROR -XepOpt:NullAway:OnlyNullMarked=true -XepOpt:NullAway:JSpecifyMode=true`.
  - **Annotation processors**: `annotationProcessorPaths` for `error_prone_core` and `nullaway`.
  - **JVM exports**: create `.mvn/jvm.config` with the `--add-exports`/`--add-opens` lines Error Prone documents for JDK 25.

  Suppressions are allowed only with a comment explaining why (constitution Workflow).
- [ ] T007 [P] Create `.githooks/pre-commit` (executable):
  1. Collect staged `*.java` files; exit 0 if there are none.
  2. Run `./mvnw -q spotless:apply -DspotlessFiles=<comma-separated regex of staged files>`.
  3. `git add` those files.
- [ ] T008 **Spike R15 (BlockHound on JDK 25)**:
  - **Setup**: add `-XX:+AllowRedefinitionToAddDeleteMethods` to the Failsafe `argLine` in `pom.xml`, and create `test/platform/BlockHoundSpikeIT.java`, which runs `Mono.delay(Duration.ofMillis(1)).doOnNext(x -> Thread.sleep(1)).block()` inside a `parallel` scheduler and asserts `BlockingOperationError` is raised.
  - **If it passes**: keep the flag and delete the spike test (T057 wires it in for all ITs).
  - **If BlockHound cannot run on JDK 25**: record the deviation in `specs/001-platform-skeleton/plan.md` Complexity Tracking, and implement the fallback from research R15 in T057 instead.
- [ ] T009 Verify Setup (`pom.xml`): `./mvnw -B verify -Ddependency-check.skip=true` succeeds on the empty project (compiles, Spotless clean, no tests yet).

**Checkpoint**: an empty project builds with formatting and static analysis enforced.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: the pieces every story's tests depend on: config binding, clock, database with two
roles, migrations, mock identity provider, real tokens in tests, generated contracts, contract
validation, and authentication.

**⚠️ CRITICAL**: no user story work can begin until this phase is complete.

### Database and migrations

- [ ] T010 [P] Create `docker/postgres/init/01-roles.sql`. It runs once, by the Postgres image on an empty data directory, as the superuser:
  - **Roles**: create `budgit_owner` (`LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE`, password from `:'owner_password'` psql variable or env default `owner` for local) and `budgit_app` (`LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE`, password default `app`).
  - **Database**: create database `budgit` owned by `budgit_owner`.
  - **Environment label**: `ALTER DATABASE budgit SET budgit.environment = '<value of env BUDGIT_ENVIRONMENT>'`. Use a small `02-environment.sh` wrapper if needed to read the env var: `local` in Compose, `test` in CI Compose and Testcontainers.
  - **Hardening**: revoke `CREATE` on schema `public` from `PUBLIC`.
- [ ] T011 [P] Create `src/main/resources/db/migration/V1__baseline.sql` (runs as `budgit_owner`; data-model §1):
  - `CREATE SCHEMA budgit;`
  - `GRANT USAGE ON SCHEMA budgit TO ${appRole};`
  - `ALTER DEFAULT PRIVILEGES FOR ROLE budgit_owner IN SCHEMA budgit GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ${appRole};`
  - `ALTER DEFAULT PRIVILEGES FOR ROLE budgit_owner IN SCHEMA budgit GRANT USAGE, SELECT ON SEQUENCES TO ${appRole};`

  It never creates roles.
- [ ] T012 Configure persistence in `src/main/resources/application.yaml` (research R13):
  - **Flyway** (owner role, startup only):
    - `spring.flyway.url/user/password` from `${BUDGIT_DB_HOST}`, `${BUDGIT_DB_PORT:5432}`, `${BUDGIT_DB_NAME:budgit}`, `${BUDGIT_DB_OWNER_USER:budgit_owner}` and `${BUDGIT_DB_OWNER_PASSWORD}`;
    - `spring.flyway.schemas=budgit`, `spring.flyway.placeholders.appRole=${BUDGIT_DB_APP_USER:budgit_app}`;
    - `spring.flyway.connect-retries=10`, `spring.flyway.connect-retries-interval=3s`.
  - **R2DBC** (app role, request path): `spring.r2dbc.url=r2dbc:postgresql://${BUDGIT_DB_HOST}:${BUDGIT_DB_PORT:5432}/${BUDGIT_DB_NAME:budgit}?schema=budgit` with `${BUDGIT_DB_APP_USER:budgit_app}` and `${BUDGIT_DB_APP_PASSWORD}`.
- [ ] T013 [P] Create `main/config/ClockConfig.java` exposing `@Bean Clock clock()` returning `Clock.systemUTC()` (constitution §III). No other code may call `Instant.now()` or `LocalDate.now()` without it.

### Configuration binding

- [ ] T014 Write `test/config/BudgitPropertiesTest.java` first. Using `ApplicationContextRunner` with `@EnableConfigurationProperties(BudgitProperties.class)` and `ValidationAutoConfiguration`, assert that startup fails, naming the property and never its value, when:
  - `budgit.environment` is missing;
  - `budgit.security.issuer-uri` is missing;
  - `budgit.security.user-client-ids` is empty;
  - `budgit.security.clock-skew` exceeds 5 min.

  Also assert that defaults apply: clock-skew 60 s, jwks-timeout 3 s, max-request-size 256 KB.
- [ ] T015 Create `main/config/BudgitProperties.java`, a `@Validated @ConfigurationProperties("budgit")` record (data-model §6):
  - **Top level**:
    - `environment`: `Environment` enum `LOCAL`/`TEST`/`PRODUCTION`, "required; must match the active profile";
    - `security`: nested record `Security`;
    - `http`: nested record `Http` with `maxRequestSize` (`DataSize`, "default 256 KB");
    - `sidecarEndpoints`: `Map<String, @NotBlank String>`, optional.
  - **`Security` fields**:
    - `issuerUri` (`URI`, required);
    - `jwkSetUri` (`URI`, optional);
    - `userClientIds` (`Set<String>`, "required, non-empty");
    - `serviceClientIds` (`Set<String>`, "required (may be empty until the first internal endpoint)");
    - `clockSkew` (`Duration`, "default 60 s, max 5 min");
    - `jwksTimeout` (`Duration`, "default 3 s").
  - **Enum location**: put `Environment` in `main/config/Environment.java`.
  - **Startup check**: an `EnvironmentProfileCheck` (`ApplicationListener<ApplicationEnvironmentPreparedEvent>` or a validator) fails startup when `budgit.environment` differs from the active profile.
  - **Done when**: T014 passes.
- [ ] T016 [P] Create `src/main/resources/application-local.yaml` and `src/main/resources/application-test.yaml`:
  - **Shared values**: `budgit.environment: LOCAL`/`TEST`, `budgit.security.issuer-uri: http://mock-oidc:8090/budgit`, `budgit.security.user-client-ids: [budgit-web]`, `budgit.security.service-client-ids: [budgit-provisioner]`.
  - **`local` only**: DB host default `localhost`, owner/app passwords `owner`/`app`.
  - **`test` only**: no DB host (tests inject it).

### Mock identity provider and test support

- [ ] T017 [P] Create `docker/mock-oidc/config.json` for mock-oauth2-server (research R12):
  - **Basics**: issuer id `budgit`; `interactiveLogin: true`; HTTP port 8090.
  - **Token callbacks** add Cognito-like claims:
    - **Client `budgit-web`** (user-facing, used with authorization code + PKCE and with the test token shortcut): `token_use: "access"`, `client_id: "budgit-web"`, `username`, `sub` = a stable test user UUID-like string, `scope: "openid"`.
    - **Client `budgit-provisioner`** (service, `client_credentials`): `token_use: "access"`, `client_id: "budgit-provisioner"`, `sub: "budgit-provisioner"`, `scope: "budgit/provision-user"`.
- [ ] T018 Create `test/support/Containers.java`, singleton Testcontainers shared by all ITs:
  - **Postgres**: `PostgreSQLContainer("postgres:17")`, with `docker/postgres/init/` copied to `/docker-entrypoint-initdb.d/` and env `BUDGIT_ENVIRONMENT=test`.
  - **Mock OIDC**: `GenericContainer("ghcr.io/navikt/mock-oauth2-server:<pinned>")` with `docker/mock-oidc/config.json`, env `SERVER_PORT=8090` and `JSON_CONFIG_PATH`, network alias `mock-oidc`.
  - **Wiring**: a `@DynamicPropertySource` helper sets `BUDGIT_DB_HOST`/port and the owner/app credentials, `budgit.security.issuer-uri` (the container's mapped issuer URL), and `budgit.security.jwk-set-uri`.

  Both containers start once per JVM.
- [ ] T019 Create `test/support/TokenFactory.java`, which obtains real signed tokens from the mock OIDC container over HTTP:
  - `userToken(String sub)` for client `budgit-web`, via the mock's token endpoint with a `sub` request parameter;
  - `serviceToken(String scope)` for client `budgit-provisioner`, via `client_credentials`;
  - `expiredUserToken()`, using the mock's `expiry` claim override;
  - `idToken()`, an OIDC ID token (`token_use` not `access`);
  - `foreignIssuerToken()`, signed by a second mock issuer id (`other`).

  There is no auth bypass anywhere (FR-020).
- [ ] T020 Create `test/support/IntegrationTest.java`, a meta-annotation:
  - `@SpringBootTest(webEnvironment = RANDOM_PORT)`, `@ActiveProfiles("test")`, `@AutoConfigureWebTestClient`;
  - an extension that wires `Containers`;
  - `management.server.port=0` (random) exposed to tests via `@LocalManagementPort`.

  Every `*IT` uses it.

### Contracts, generation and validation

- [ ] T021 [P] Copy `specs/001-platform-skeleton/contracts/budgit-api.yaml` to `src/main/resources/openapi/public/budgit-api.yaml`, and `contracts/budgit-internal-api.yaml` to `src/main/resources/openapi/internal/budgit-internal-api.yaml`. Fix the internal file's `$ref`s to `../public/budgit-api.yaml#/...`. Both have `info.version: 0.1.0`.
- [ ] T022 Configure `openapi-generator-maven-plugin` in `pom.xml` with two executions in `generate-sources` (research R3):
  - **Shared options**:
    - `generatorName=spring`, `library=spring-boot`, `reactive=true`, `interfaceOnly=true`, `useJakartaEe=true`, `useBeanValidation=true`;
    - the Boot 4 flag the generator version provides;
    - `skipDefaultInterface=true`, `openApiNullable=false`;
    - `schemaMappings`/`importMappings` `Problem=org.springframework.http.ProblemDetail`.
  - **Per execution**: the public spec → `com.danekoval.budgit.generated.publicapi`; the internal spec → `com.danekoval.budgit.generated.internalapi`.
  - **Done when**: `./mvnw compile` succeeds.
- [ ] T023 Write `test/support/ContractValidatorSelfTest.java` first, the R4 validator self-tests. Using a small test-only OpenAPI 3.1 file `src/test/resources/openapi/validator-selftest.yaml` (one operation returning an object with a required `name`, a nullable `note` declared `type: [string, "null"]`, and a `const` field), assert that `ContractValidatingFilter`:
  - **passes** a valid response, including `note: null`;
  - **fails** on: a missing required field; `note: 123`; a wrong `const` value; an undeclared status code; an undeclared content type; a missing required response header.

  If any of these can't be detected by `openapi-request-validator-core` 3.x, switch to the fallback in research R4 (swagger-parser + networknt json-schema-validator) and record that in research.md.
- [ ] T024 Create `test/support/ContractValidatingFilter.java`, an `ExchangeFilterFunction` (research R4):
  - **Validation**: builds `SimpleRequest`/`SimpleResponse` from each `WebTestClient` exchange (method, path, query, headers, request body, status, response headers, response body) and validates them with an `OpenApiInteractionValidator` loaded from the public contract, plus any test-only contracts registered for path prefixes. Any report message at ERROR level throws `AssertionError` listing all messages.
  - **Body handling**: it reads the response body once and re-emits it, so tests can still assert on it.
  - **Registration**: register it on every `WebTestClient` via a `WebTestClientBuilderCustomizer` bean in `test/support/ContractValidationConfig.java`.
  - **Done when**: T023 passes.

### Authentication (two caller kinds)

- [ ] T025 Write `test/platform/AuthenticationIT.java` first, asserting (FR-019, Story 2 #1 and #9):
  - **401 cases**: requests to `/api/v1/anything` with:
    - no token;
    - an expired token;
    - a `foreignIssuerToken`;
    - an `idToken`;
    - a `serviceToken`.

    Each gets 401, and the bodies are identical apart from `requestId`.
  - **Valid user token**: `/api/v1/anything` gets 404 (authenticated, no route).
  - **Internal path**: a `userToken` on `/internal/v1/anything` gets 401; a `serviceToken` gets 404.
- [ ] T026 Create `main/config/SecurityConfig.java` (research R9):
  - **`ReactiveJwtDecoder`**: `NimbusReactiveJwtDecoder` from `budgit.security.jwk-set-uri`, or the issuer's discovered JWKS. Its `WebClient` has connect timeout 2 s and response timeout `budgit.security.jwks-timeout`.
  - **Validators**: issuer, `JwtTimestampValidator` with `clockSkew` and the injected `Clock`, `token_use == "access"`.
  - **Two ordered `SecurityWebFilterChain`s**:
    - `/internal/**`: requires `client_id` ∈ `serviceClientIds`, and authorizes per-endpoint scopes via `hasAuthority("SCOPE_…")`. No endpoints exist yet.
    - Everything else on the app port: requires `client_id` ∈ `userClientIds`.
  - **Other chain settings**: both are stateless, with CSRF, form login, HTTP basic and the default cache-control headers disabled.
  - **Done when**: T025 passes. A placeholder `ServerAuthenticationEntryPoint` returning a bare 401 is fine until T046.

**Checkpoint**: the app starts in `test` against Testcontainers Postgres (migrated, two roles) and
the mock OIDC server; tests obtain real tokens; every test exchange is contract-validated;
authentication works for both caller kinds.

---

## Phase 3: User Story 1 - Run the service locally and see it healthy (Priority: P1) 🎯 MVP

**Goal**: a developer clones, builds, starts the stack with one command and sees the API healthy on
the management port.

**Independent Test**: quickstart §1–§3. `./mvnw verify` passes. `docker compose up --build --wait`
brings up `postgres`, `mock-oidc` and `api`; the readiness and liveness probes return `UP`; `info`
shows build info; `env` returns 404; `/actuator/health` on 8080 is not served.

### Tests for User Story 1 ⚠️

- [ ] T027 [P] [US1] Write `test/platform/ManagementEndpointsIT.java` (FR-026, Story 1 #3–4):
  - **Management port**: `/actuator/health/liveness` and `/actuator/health/readiness` return 200 `{"status":"UP"}`; `/actuator/info` returns `build.version` and `git.commit.id`; `/actuator/env`, `/actuator/beans` and `/actuator/metrics` return 404.
  - **App port**: `/actuator/health` without a token returns 401; with a user token, 404.
- [ ] T028 [P] [US1] Write `test/platform/MigrationIT.java`: after startup, schema `budgit` exists, and `flyway_schema_history` shows `V1__baseline` applied successfully, as `budgit_owner`.

### Implementation for User Story 1

- [ ] T029 [US1] Configure Actuator in `src/main/resources/application.yaml`:
  - `management.server.port: 8081`;
  - `management.endpoints.web.exposure.include: health,info`;
  - `management.endpoint.health.probes.enabled: true`;
  - `management.endpoint.health.group.readiness.include: readinessState,r2dbc`;
  - `management.endpoint.health.group.liveness.include: livenessState`;
  - `management.info.git.mode: simple`.

  Also add `git-commit-id-maven-plugin` and the `spring-boot-maven-plugin` `build-info` goal to `pom.xml`. Done when T027 passes.
- [ ] T030 [P] [US1] Create `Dockerfile` (research R17):
  - **Build stage**: `eclipse-temurin:25-jdk@sha256:<digest>`, runs `./mvnw -B -DskipTests package`, then `java -Djarmode=tools -jar target/budget-api-*.jar extract --layers --launcher --destination /app`.
  - **Runtime stage**: `eclipse-temurin:25-jre@sha256:<digest>`, non-root user `budgit` (uid 10001). It copies the layers `dependencies`, `spring-boot-loader`, `snapshot-dependencies` and `application` in that order.
  - **Run settings**: `EXPOSE 8080 8081`; `ENTRYPOINT ["java","org.springframework.boot.loader.launch.JarLauncher"]`; no `HEALTHCHECK`.
  - **JVM warnings**: not suppressed; any JVM flag added needs a comment saying why (research R17).
- [ ] T031 [US1] Create `compose.yaml` (research R18):
  - **`postgres`**: `postgres:17`, env `BUDGIT_ENVIRONMENT=local`, mounts `./docker/postgres/init:/docker-entrypoint-initdb.d:ro` and volume `pgdata`, port 5432, healthcheck `pg_isready`.
  - **`mock-oidc`**: mock-oauth2-server, env `SERVER_PORT=8090`, `JSON_CONFIG_PATH=/config.json`, mounts `./docker/mock-oidc/config.json`, port `8090:8090`, hostname `mock-oidc`.
  - **`api`**: `build: .`, `SPRING_PROFILES_ACTIVE=local`, `BUDGIT_DB_HOST=postgres`, and owner/app passwords; ports `8080:8080` and `8081:8081`; `depends_on` postgres (healthy) and mock-oidc (started); healthcheck on `http://localhost:8081/actuator/health/readiness`.
  - **`ui`**: profile `ui`, image `ghcr.io/<owner>/budget-app:<release>` (placeholder comment until `budget-app` publishes a release), `depends_on` api.
- [ ] T032 [P] [US1] Create `dev/seed/000_guard.sql` (FR-029, research R18):
  ```sql
  DO $$ BEGIN
    IF current_setting('budgit.environment', true) IS DISTINCT FROM 'local' THEN
      RAISE EXCEPTION 'Dev seed refused: database is not marked local';
    END IF;
  END $$;
  ```
  Add `dev/seed/README.md` explaining that later specs add numbered files after it.
- [ ] T033 [US1] Add the `seed` service to `compose.yaml`:
  - profile `seed`, image `postgres:17`;
  - mounts `./dev/seed:/seed:ro`;
  - `depends_on` postgres (healthy);
  - entrypoint runs each `/seed/*.sql` in order with `psql -v ON_ERROR_STOP=1 -h postgres -U budgit_app -d budgit` and exits non-zero on the first failure; env `PGPASSWORD=app`.
- [ ] T034 [US1] Run `specs/001-platform-skeleton/quickstart.md` §1–§3 and §8 manually and fix any gaps:
  - `./mvnw verify`;
  - `docker compose up --build --wait`;
  - the curl checks;
  - the seed: local succeeds, and the `-p budgit-ci` guard check aborts.

**Checkpoint**: US1 complete. The skeleton runs locally and is healthy.

---

## Phase 4: User Story 2 - Every response follows the platform contract (Priority: P1)

**Goal**: every response, from any source, has `X-Request-Id` and `Cache-Control: no-store`, and
every error is Problem Details with a stable code and no leaked data.

**Independent Test**: quickstart §4, plus `ErrorResponsesIT`, `RequestIdIT` and `CacheControlIT`
passing.

### Tests for User Story 2 ⚠️

- [ ] T035 [P] [US2] Write `test/platform/RequestIdIT.java` (FR-014, Story 2 #2–3):
  - an incoming `X-Request-Id: demo-123` is echoed in the response header and the Problem body's `requestId`;
  - a missing header gets a generated UUID;
  - each of these invalid values is replaced with a generated UUID: `bad id`, a 65-character value, `a\nb`, an empty string.
- [ ] T036 [P] [US2] Write `test/platform/ErrorResponsesIT.java` (FR-011, FR-012, Story 2 #1 and #5–8; contracts/problem-codes.md). With a user token:
  - **Not found**: an unknown path gives 404 `NOT_FOUND`.
  - **Wrong method**: `DELETE` on a GET-only test route gives 405 `METHOD_NOT_ALLOWED`.
  - **`Accept: application/xml`** gives 406 `NOT_ACCEPTABLE`.
  - **Bad request bodies** on a test route:
    - `Content-Type: text/plain` gives 415 `UNSUPPORTED_MEDIA_TYPE`;
    - a body over 256 KB gives 413 `PAYLOAD_TOO_LARGE`;
    - malformed JSON gives 400 `VALIDATION_FAILED`;
    - an unknown JSON field gives 400 `VALIDATION_FAILED` with `errors[{field, code:"UNKNOWN_FIELD"}]`.
  - **Unexpected error**: a test route that throws `IllegalStateException("secret-detail")` gives 500 `INTERNAL_ERROR`, and the body does not contain `secret-detail` or a stack trace.
  - **Every case**: `Content-Type: application/problem+json`, and the body has `type`, `title`, `status`, `code` and `requestId`, with no `detail` copied from an exception.

  The test-only routes live in `test/platform/support/ErrorProbeController.java`, with a matching test-only contract `src/test/resources/openapi/error-probe.yaml` registered with the validator.
- [ ] T037 [P] [US2] Write `test/platform/CacheControlIT.java` (FR-013, SC-002): exactly `Cache-Control: no-store` on a 200 test route, a 401, a 404, a 415, a 500 and a 503.
- [ ] T038 [P] [US2] Write `test/platform/ServiceUnavailableIT.java` (Story 2 #7):
  - **Database down**: with the R2DBC pool pointed at a stopped Postgres container (a separate throwaway container for this test), a DB-backed test route returns 503 `SERVICE_UNAVAILABLE`.
  - **Shutting down**: after publishing `AvailabilityChangeEvent` with `ReadinessState.REFUSING_TRAFFIC`, any app-port request returns 503 and the readiness endpoint returns 503.

### Implementation for User Story 2

- [ ] T039 [P] [US2] Create `main/common/ProblemCode.java`, an enum with the exact codes from `contracts/problem-codes.md`:
  - `VALIDATION_FAILED`, `UNAUTHENTICATED`, `NOT_FOUND`, `METHOD_NOT_ALLOWED`, `NOT_ACCEPTABLE`, `STALE_VERSION`;
  - `ID_CONFLICT`, `PAYLOAD_TOO_LARGE`, `UNSUPPORTED_MEDIA_TYPE`, `BUSINESS_RULE_VIOLATION`, `RATE_LIMITED`;
  - `INTERNAL_ERROR`, `UPSTREAM_FAILED`, `SERVICE_UNAVAILABLE`, `UPSTREAM_TIMEOUT`.

  Each constant carries its HTTP status and a `type` URI `https://budgit.app/problems/<kebab-case-code>`.
- [ ] T040 [P] [US2] Create the platform exceptions in `main/common/`:
  - `NotFoundException`, `StaleVersionException`, `IdConflictException`, `BusinessRuleException` (optional specific `ProblemCode`), `UpstreamFailedException` and `UpstreamTimeoutException`;
  - all extend `PlatformException`, which carries a `ProblemCode`;
  - messages are for logs only and never reach response bodies.
- [ ] T041 [US2] Create `main/api/filter/RequestIdWebFilter.java` (research R8):
  - **Ordering**: a `WebFilter` at `Ordered.HIGHEST_PRECEDENCE`.
  - **Request ID**: accepts `X-Request-Id` matching `^[A-Za-z0-9._-]{1,64}$`, otherwise generates `UUID.randomUUID().toString()`.
  - **Where it's stored**: as exchange attribute `budgit.requestId`, in the Reactor `Context`, and as tracing baggage `requestId` (via `Tracer.createBaggageInScope` / the Micrometer baggage API).
  - **Response header**: `X-Request-Id` set via `beforeCommit`.

  Create `main/common/RequestId.java` as a record holding the validated value. Done when T035 passes.
- [ ] T042 [US2] Create `main/api/filter/CacheControlWebFilter.java`, which sets `Cache-Control: no-store` via `beforeCommit` on every app-port response. Spring Security's own cache headers are already disabled in T026.
- [ ] T043 [US2] Create `main/api/ProblemDetailsFactory.java`, which builds a `ProblemDetail` from a `ProblemCode` and optional field errors:
  - **Fields**: `type`, a fixed generic `title` per status, `status`, and properties `code` and `requestId` (from the exchange attribute);
  - **Per-field errors**: `errors` as `[{field, code}]`, never including rejected values;
  - **Never set**: `detail`.
- [ ] T044 [US2] Create `main/api/ProblemDetailsAdvice.java`, a `@RestControllerAdvice` extending `ResponseEntityExceptionHandler`. It maps:
  - `PlatformException` subclasses to their codes;
  - `WebExchangeBindException`/validation to 400 `VALIDATION_FAILED` with field errors;
  - `ServerWebInputException` (including unknown fields) to 400, with `UNKNOWN_FIELD` field errors;
  - `UnsupportedMediaTypeStatusException` to 415;
  - `DataBufferLimitException` to 413;
  - `MethodNotAllowedException` to 405;
  - `NotAcceptableStatusException` to 406;
  - `DataAccessResourceFailureException`/R2DBC connection failures to 503;
  - anything else to 500 `INTERNAL_ERROR`, logged at ERROR with the stack trace in logs only.

  It uses `ProblemDetailsFactory` throughout.
- [ ] T045 [US2] Create `main/api/ProblemWebExceptionHandler.java`, a `WebExceptionHandler` with `@Order(-2)` that writes Problem Details (via `ProblemDetailsFactory` and the codec) for errors outside controllers: `ResponseStatusException` 404 for unmatched routes, plus any remaining exception as 500.
  - **Same handling as the controller path**: it also handles the `ResponseStatusException` subclasses for 405, 406, 413 and 415 raised before a controller is selected, with the same mapping as `ProblemDetailsAdvice`.
  - **Headers**: it sets `X-Request-Id` and `Cache-Control: no-store` too, in case the filters didn't run.
- [ ] T046 [US2] Create `main/api/ProblemAuthenticationEntryPoint.java`, a `ServerAuthenticationEntryPoint` that writes 401 `UNAUTHENTICATED` Problem Details. The body is the same for every cause; the cause is logged at DEBUG without the token. Wire it into both chains in `main/config/SecurityConfig.java`.
  - **JWKS failures**: map `AuthenticationServiceException` (e.g. the JWKS is unreachable) to 503 `SERVICE_UNAVAILABLE`, failing closed.
  - **Done when**: T025's identical-body assertion passes.
- [ ] T047 [US2] Create `main/api/filter/ReadinessGateWebFilter.java`, which returns 503 `SERVICE_UNAVAILABLE` Problem Details for app-port requests while `ApplicationAvailability.getReadinessState()` is `REFUSING_TRAFFIC`. Set `server.shutdown: graceful` and `spring.lifecycle.timeout-per-shutdown-phase: 20s` in `src/main/resources/application.yaml`. Done when T038 passes.
- [ ] T048 [US2] Configure in `src/main/resources/application.yaml`:
  - `spring.jackson.deserialization.fail-on-unknown-properties: true`;
  - `spring.http.codecs.max-in-memory-size: ${budgit.http.max-request-size}`;
  - `spring.webflux.problemdetails.enabled: true`.

  Done when T036 and T037 pass.

**Checkpoint**: US2 complete. Every response follows the platform contract.

---

## Phase 5: User Story 3 - The contract and shared fixtures are published for the UI (Priority: P1)

**Goal**: contract lint and the breaking-change check run in `verify`; each merge to `main`
publishes a versioned release containing only the public contract and the shared fixtures.

**Independent Test**:
- `./mvnw verify` runs Redocly lint and oasdiff (report-only while `0.x`);
- a deliberately broken contract fails lint;
- the publish workflow, run on a test branch via `workflow_dispatch`, produces `contract-v0.1.<run>` with `budgit-api-0.1.<run>.yaml` (no `/internal` paths) and `shared-fixtures-0.1.<run>.tar.gz`.

### Tests for User Story 3 ⚠️

- [ ] T049 [P] [US3] Create `scripts/test/openapi-breaking-check.test.sh`, a shell test of the breaking-check script using fixture files in `scripts/test/fixtures/`. It asserts the exit codes for:
  - no baseline → 0, with a "skipped" message;
  - a `0.x` breaking change → 0, with the report printed;
  - a `1.x` breaking change → non-zero;
  - a `1.x` → `2.0.0` breaking change → 0, skipped;
  - a non-breaking `1.x` change → 0.

  Run it from `verify` via `exec-maven-plugin`.

### Implementation for User Story 3

- [ ] T050 [P] [US3] Create `redocly.yaml` (research R5):
  - extends `recommended`;
  - `operation-operationId: error`;
  - a configurable rule requiring kebab-case path segments;
  - a configurable rule requiring public paths to start with `/api/v1`;
  - a configurable rule requiring every operation to declare `401` and `500` responses referencing `#/components/responses/*`.

  Apply it to both `src/main/resources/openapi/public/budgit-api.yaml` and `src/main/resources/openapi/internal/budgit-internal-api.yaml`. The internal spec's path-prefix rule is `/internal/v1`.
- [ ] T051 [US3] Add an `exec-maven-plugin` execution in `pom.xml` (phase `validate`): `docker run --rm -v ${project.basedir}:/spec redocly/cli:<pinned> lint --config /spec/redocly.yaml` on both contracts. The build fails on lint errors.
- [ ] T052 [US3] Create `scripts/openapi-breaking-check.sh` (research R5, constitution §IV v1.1.0):
  1. Fetch the baseline with `git show origin/main:src/main/resources/openapi/public/budgit-api.yaml` into `target/openapi-baseline.yaml`; if there's none, print "breaking-change check skipped: no baseline" and exit 0.
  2. Read both `info.version`s.
  3. If the major version increased, print a "skipped: major version bump" message and exit 0.
  4. Run `docker run --rm -v ...:/w tufin/oasdiff:<pinned> breaking /w/base.yaml /w/head.yaml --fail-on ERR`.
  5. If the head version's major is 0, print the report and exit 0 (report-only before 1.0.0); otherwise exit with oasdiff's status.

  Add an `exec-maven-plugin` execution in `pom.xml` (phase `validate`) that runs it. Done when T049 passes.
- [ ] T053 [P] [US3] Create `src/test/resources/shared-fixtures/README.md`. It explains that the shared fixtures (JSON inputs and expected outputs run by both the API and the UI, constitution §VII) are published with each contract release, and that the folder is empty until the first domain rule.
- [ ] T054 [US3] Create `.github/workflows/publish-contract.yml` (research R6):
  - **Triggers**: on `workflow_run` of `ci` completed with success on `main`, and `workflow_dispatch`; permissions `contents: write`.
  - **Steps**:
    1. Check out the repo.
    2. `docker run redocly/cli bundle` the public contract to `budgit-api.yaml`.
    3. Compute the version `{major}.{minor}.${{ github.run_number }}` from the committed `info.version`, and rewrite `info.version` in the bundle (with `yq`).
    4. Assert the bundle has no path starting with `/internal`; fail if it does.
    5. `tar czf shared-fixtures-<v>.tar.gz -C src/test/resources shared-fixtures`.
    6. `gh release create contract-v<v>` with both assets, `budgit-api-<v>.yaml` and `shared-fixtures-<v>.tar.gz`.

**Checkpoint**: US3 complete. The contract is checked on every build and published on every merge.

---

## Phase 6: User Story 4 - Merges are blocked unless every constitution gate passes (Priority: P2)

**Goal**: `./mvnw verify` enforces every gate; GitHub Actions runs it plus image scanning and the
E2E stack; scheduled scans and Dependabot are on.

**Independent Test**: open one throwaway PR per gate that violates it (SC-004 checklist in T085);
each is blocked; a clean PR passes `verify`, `image` and `e2e`.

### Tests for User Story 4 ⚠️

- [ ] T055 [US4] Create `test/architecture/ArchitectureTest.java` (`@AnalyzeClasses(packages = "com.danekoval.budgit", importOptions = DoNotIncludeTests.class)`) with the layering rules (constitution §VII):
  - `..domain..` must not depend on `org.springframework..`, `reactor..`, `..persistence..`, `..api..` or `..generated..`;
  - `..api..` must not depend on `..persistence..`;
  - `..generated..` classes must not be used from `..domain..`;
  - no classes use `lombok..` or `org.mapstruct..`;
  - no class calls `java.time.Instant.now()`, `LocalDate.now()` or `LocalDateTime.now()` without a `Clock` argument;
  - no field of type `ThreadLocal` in `src/main`.

  Create `src/test/resources/archunit.properties` with `archRule.failOnEmptyShould=false`.

### Implementation for User Story 4

- [ ] T056 [US4] Configure JaCoCo in `pom.xml`:
  - `prepare-agent` for unit tests, and `prepare-agent-integration` for ITs;
  - `merge` and `report`;
  - `check` in `verify` with a rule on `PACKAGE` elements `com.danekoval.budgit.domain*`, `LINE` covered ratio minimum `0.85` (constitution §IX).
- [ ] T057 [US4] Wire BlockHound for ITs:
  - **If T008 passed**: keep `blockhound-junit-platform` on the Failsafe classpath only, with `-XX:+AllowRedefinitionToAddDeleteMethods` in the Failsafe `argLine`, and add `test/support/BlockHoundConfig.java` (a `BlockHoundIntegration`) that allows known-safe framework internals only with a comment each.
  - **If T008 failed**: implement the R15 fallback instead:
    - an ArchUnit rule in `test/architecture/ArchitectureTest.java` banning calls to `Mono.block*`, `Flux.block*`, `CompletableFuture.get`, `Future.get`, `Thread.sleep` and `java.io.InputStream.read` in `src/main`, except in methods annotated `@BlockingAllowed(reason)` (create `main/common/BlockingAllowed.java`);
    - an IT asserting request handling runs on `reactor-http-nio-*` threads.

    Record the deviation in plan.md Complexity Tracking.
- [ ] T058 [P] [US4] Configure `dependency-check-maven` in `pom.xml`:
  - goal `check` in `verify`;
  - `failBuildOnCVSS=7`;
  - `nvdApiKeyEnvironmentVariable=NVD_API_KEY`;
  - `suppressionFile=dependency-check-suppressions.xml`;
  - `skip` property `dependency-check.skip`.

  Create `dependency-check-suppressions.xml` with a header comment requiring `<notes>` (why) and an `until="YYYY-MM-DD"` attribute (re-check date) on every `<suppress>`. It has no entries yet.
- [ ] T059 [P] [US4] Create `compose.ci.yaml`, overrides only:
  - **`postgres`**: `BUDGIT_ENVIRONMENT=test`.
  - **`api`**: `SPRING_PROFILES_ACTIVE=test` and `image: ${BUDGIT_API_IMAGE:-budget-api:ci}`, plus the host, port and credential variables the `test` profile needs (set explicitly, since the `test` profile has none).
- [ ] T060 [P] [US4] Create `scripts/e2e-smoke.sh`, the API smoke suite against the running stack, using `curl` and `jq` (research R20). It exits non-zero on the first failed check:
  - the readiness probe on 8081 returns `UP`;
  - `/api/v1/anything` with no token gives 401, `application/problem+json`, `code` `UNAUTHENTICATED`, `Cache-Control: no-store`, and an `X-Request-Id` header matching the body;
  - a user token obtained from `http://mock-oidc:8090/budgit/token` gives 404 `NOT_FOUND`;
  - a service token on `/api/v1/anything` gives 401.
- [ ] T061 [US4] Create `.github/workflows/ci.yml` (research R20):
  - **Triggers**: `pull_request` and `push` to `main`.
  - **`verify` job**: ubuntu-latest; Temurin 25 via `actions/setup-java` with Maven cache; restore/save the NVD data cache (`~/.m2/repository/org/owasp/dependency-check-data`); `git fetch origin main`; `./mvnw -B verify` with `NVD_API_KEY` from secrets.
  - **`image` job**:
    1. Build the image as `budget-api:ci` and save it as an artifact.
    2. Run `aquasec/trivy image --severity HIGH,CRITICAL --exit-code 1 --ignorefile .trivyignore.yaml budget-api:ci`.
    3. Run the production-shaped smoke from T083.
  - **`e2e` job** (needs `image`):
    1. Load the image and add `127.0.0.1 mock-oidc` to `/etc/hosts`.
    2. `docker compose -f compose.yaml -f compose.ci.yaml up -d --wait postgres mock-oidc api`, then `scripts/e2e-smoke.sh`.
    3. Always dump `docker compose logs` on failure.

    Add a comment that the UI's E2E suite is added here once `budget-app` publishes a release.
- [ ] T062 [P] [US4] Create `.trivyignore.yaml` with a header comment: every entry needs `statement` (why) and `expired_at` (re-check date). It has no entries yet.
- [ ] T063 [P] [US4] Create `.github/workflows/security-scan.yml`:
  - **Triggers**: `schedule` cron daily (`0 6 * * *`) and `workflow_dispatch`, on `main`.
  - **Steps**:
    - `./mvnw -B -DskipTests verify` limited to the dependency-check goal (`./mvnw -B dependency-check:check`) with `NVD_API_KEY`;
    - build the image and run the Trivy scan as in `ci.yml`;
    - fail visibly on findings.
- [ ] T064 [P] [US4] Create `.github/dependabot.yml`: weekly updates for the `maven` (root), `github-actions` (root) and `docker` (root `Dockerfile`) ecosystems; open-PR limit 5 each.

**Checkpoint**: US4 complete. Every constitution gate runs locally and in CI.

---

## Phase 7: User Story 5 - Shared mechanisms for ownership and concurrency are ready (Priority: P2)

**Goal**: reusable owner-scoped persistence, optimistic version checks and idempotent creates,
enforced by ArchUnit and proven end to end with a test-only record.

**Independent Test**: `OwnershipAndVersionIT` and `DatabaseRolesIT` pass (quickstart §5).
ArchitectureTest fails if a persistence method without `UserId`/`BudgetId` is added.

### Tests for User Story 5 ⚠️

- [ ] T065 [P] [US5] Create `src/test/resources/db/test-schema.sql` (data-model §3). It's run by test setup as `budgit_owner` after Flyway, never as a migration:
  ```sql
  CREATE TABLE IF NOT EXISTS budgit.owned_widget (
      id uuid PRIMARY KEY,
      user_id uuid NOT NULL,
      name text NOT NULL CHECK (char_length(name) BETWEEN 1 AND 100),
      version integer NOT NULL DEFAULT 1,
      created_at timestamptz NOT NULL,
      updated_at timestamptz NOT NULL);
  CREATE INDEX IF NOT EXISTS owned_widget_user_id_idx ON budgit.owned_widget (user_id);
  ```
  Extend `test/support/Containers.java` to run it once over JDBC as the owner.
- [ ] T066 [P] [US5] Create `src/test/resources/openapi/test-widgets.yaml`, an OpenAPI 3.1 test-only contract that `$ref`s the public components (`Problem`, error responses, headers):
  - **Operations**:
    - `POST /api/v1/test-widgets`: body `{id?: uuid, name: string 1–100}`; responses 201 and 200 (`Widget`), 400, 401, 409, 500.
    - `GET /api/v1/test-widgets/{id}`: 200, 401, 404, 500.
    - `PATCH /api/v1/test-widgets/{id}`: body `{name?: string 1–100, version: integer ≥ 1}`; responses 200, 400, 401, 404, 409, 500.
    - `DELETE /api/v1/test-widgets/{id}?version=`: 204, 401, 404, 409, 500.
  - **`Widget` schema**: `{id, name, version, createdAt, updatedAt}`.

  Register it with `ContractValidatingFilter` for the prefix `/api/v1/test-widgets`.
- [ ] T067 [US5] Write `test/testwidget/OwnershipAndVersionIT.java` (FR-015–FR-017, Story 5 #1–3 and #5; contracts/problem-codes.md "Create with a client-supplied ID"):
  - **Creates**:
    - `POST` without `id` → 201 with a generated id and version 1;
    - `POST` with a new `id` → 201;
    - repeating the same `POST` (same `id`, same user, even with a different `name`) → 200 with the original record unchanged;
    - `POST` with user A's `id` as user B → 409 `ID_CONFLICT`.
  - **Reads**: `GET` own → 200; `GET` another user's and `GET` missing → 404 with bodies equal apart from `requestId`.
  - **Updates**:
    - `PATCH` with the current version → 200 and `version + 1`;
    - `PATCH` with a stale version → 409 `STALE_VERSION`, and the row is unchanged in the DB;
    - `PATCH` on another user's record → 404.
  - **Deletes**: `DELETE` with the stale version → 409; with the current version → 204; another user's → 404.
- [ ] T068 [P] [US5] Write `test/platform/DatabaseRolesIT.java` (FR-018, Story 5 #4). Over a raw R2DBC connection using the app's configured credentials:
  - `SELECT rolsuper FROM pg_roles WHERE rolname = current_user` → false;
  - `current_user` is not the owner of schema `budgit`;
  - `CREATE TABLE budgit.x (id int)` fails with SQLSTATE `42501` (insufficient privilege);
  - `flyway_schema_history.installed_by` = `budgit_owner`.

### Implementation for User Story 5

- [ ] T069 [P] [US5] Create the typed IDs in `main/common/`: `UserId.java` and `BudgetId.java` (records wrapping a non-null `UUID`, with `of(String)` and `toString()`), plus `UnscopedQuery.java` (`@Target(METHOD) @Retention(RUNTIME)` annotation with a required `String reason()`).
- [ ] T070 [US5] Add the ownership and transaction rules to `test/architecture/ArchitectureTest.java` (research R13):
  - **Owner parameter**: every public method of classes or interfaces in `..persistence..`, except those annotated `@UnscopedQuery`, must declare a parameter of type `UserId` or `BudgetId` (custom `ArchCondition`).
  - **Non-blank reason**: every `@UnscopedQuery` has a non-blank `reason`.
  - **No inherited unscoped methods**: no class in `..persistence..` may be assignable to `org.springframework.data.repository.CrudRepository` or `ReactiveCrudRepository`.
  - **Transactions**: every public method of `@Service` classes in `..service..` that depend on `..persistence..` must be annotated `@Transactional` (or the class must be), and must return `Mono` or `Flux`.

  Verify each rule fails against a temporary violating class, then delete that class.
- [ ] T071 [US5] Create `main/persistence/VersionedUpdates.java`, a component using `DatabaseClient` (research R13):
  - **Signature**: `<T> Mono<T> update(String table, UserId owner, UUID id, int expectedVersion, Map<String, Object> changes, Function<Readable, T> mapper)`.
  - **Update statement**: `UPDATE budgit.<table> SET <changes>, version = version + 1, updated_at = :now WHERE id = :id AND user_id = :owner AND version = :expected RETURNING *`, with `:now` from the injected `Clock`.
  - **On zero rows**: run `SELECT 1 FROM budgit.<table> WHERE id = :id AND user_id = :owner`. If the row exists → `StaleVersionException`; otherwise → `NotFoundException`.
  - **`delete`**: same contract, with `DELETE … RETURNING id`.
  - **Safety**: table and column names come only from code constants, never from input.
- [ ] T072 [US5] Create `main/persistence/IdempotentCreates.java`, a component using `DatabaseClient`:
  - **Signature**: `<T> Mono<CreateResult<T>> create(String table, UserId owner, UUID id, Map<String, Object> columns, Function<Readable, T> mapper)`.
  - **Insert**: `INSERT … ON CONFLICT (id) DO NOTHING RETURNING *`. A row returned → `CreateResult.created(row)`.
  - **On conflict**: `SELECT * … WHERE id = :id AND user_id = :owner`. Found → `CreateResult.existing(row)`; otherwise → `IdConflictException`.
  - **`CreateResult<T>`**: a record `(T value, boolean created)` in `main/persistence/CreateResult.java`.

  `created_at`/`updated_at` come from the injected `Clock`.
- [ ] T073 [US5] Create the test-only widget slice in `test/testwidget/`:
  - **`WidgetRepository.java`**: implements `Repository` or uses `DatabaseClient`; every method takes `UserId`.
  - **`WidgetService.java`**: `@Service`, `@Transactional` public methods returning `Mono`, using `VersionedUpdates`/`IdempotentCreates`.
  - **`WidgetController.java`**: `@RestController` for the T066 contract; it returns 201 or 200 from `CreateResult.created()`.
  - **Request records**: `CreateWidgetRequest`/`UpdateWidgetRequest` with Bean Validation `@Size(min=1,max=100)` on `name` and `@Min(1)` on `version`.
  - **Owner**: resolve `UserId` from the JWT `sub` claim, using a test-only deterministic UUID-from-sub mapping, since user provisioning comes in the next spec.

  Done when T067 passes.

**Checkpoint**: US5 complete. Ownership and concurrency are proven and enforced.

---

## Phase 8: User Story 6 - The service is observable without leaking financial data (Priority: P2)

**Goal**: JSON logs with request and trace correlation, one completion line per request, OTel
traces and metrics with safe tags, and an automated privacy test.

**Independent Test**: `LogPrivacyIT` and `ObservabilityIT` pass; quickstart §6 shows the request's
lines with `requestId`/`trace.id`, and lists any non-JSON lines for review.

### Tests for User Story 6 ⚠️

- [ ] T074 [P] [US6] Write `test/platform/LogPrivacyIT.java` (FR-024, SC-006, research R16):
  - **Requests**: with `OutputCaptureExtension`, an in-memory OTel span exporter bean (`test/support/InMemorySpansConfig.java`) and a `SimpleMeterRegistry`, send requests whose headers, query string and JSON body contain these sentinels:
    - `987654321` (amount);
    - `SENTINEL-PAYEE-7f1c`;
    - `SENTINEL-MEMO-7f1c`;
    - `SENTINEL-ACCOUNT-7f1c`;
    - `sentinel.user@example.com`;
    - the bearer token string itself.

    Send them to a 200 test-widget route, a 400 route, a 401 and a 500 route.
  - **No sentinel anywhere**: in captured stdout, in any span attribute or name, or in any meter tag.
  - **Format**: every captured stdout line parses as JSON.
  - **Correlation**: every line containing one of the test's request IDs has `requestId` and `trace.id`.
- [ ] T075 [P] [US6] Write `test/platform/ObservabilityIT.java` (FR-023, FR-025):
  - **Completion line**: exactly one INFO line with `message` "Request completed", plus `http.method`, `url.template` (e.g. `/api/v1/test-widgets/{id}`, never the actual id), `http.status`, `duration.ms`, `requestId` and `trace.id`.
  - **Spans**: a server span exists per request.
  - **Metrics**: `http.server.requests` has only allow-listed tag keys.
  - **Tag allow-list**: a meter registered by test code with tag `userId` has that tag removed.

### Implementation for User Story 6

- [ ] T076 [US6] Configure logging in `src/main/resources/application.yaml`:
  - `logging.structured.format.console: ecs` (structured logging to stdout only, no file appenders);
  - `spring.main.banner-mode: off`;
  - `spring.reactor.context-propagation: auto`;
  - `management.tracing.baggage.remote-fields: []`;
  - `management.tracing.baggage.correlation.fields: requestId`;
  - no `logging.level` entries for codecs, wiretap or `org.springframework.web.reactive.function.client`.
- [ ] T077 [US6] Configure tracing and metrics in `src/main/resources/application.yaml`, `application-local.yaml` and `application-test.yaml`:
  - **Sampling**: `management.tracing.sampling.probability: 1.0`.
  - **Export**: OTLP tracing and metrics export disabled in `local` and `test` (spans still created); in `production`, endpoints come from config (T081).

  Use the Boot 4.1 property names, checking them in the Boot reference.
- [ ] T078 [US6] Create `main/config/ObservabilityConfig.java` with a `MeterFilter` bean that removes any tag whose key is not in `{method, uri, status, outcome, exception, error}` (constitution §XI). Done when T075's allow-list assertion passes.
- [ ] T079 [US6] Create `main/api/filter/RequestLoggingWebFilter.java` (research R16):
  - **What it logs**: once per request, on completion (`doFinally`/`then`, inside the request's Reactor context), one INFO line "Request completed" with structured fields `http.method`, `url.template`, `http.status` and `duration.ms` (measured with the injected `Clock`). The route pattern comes from `RouterFunctions.MATCHING_PATTERN_ATTRIBUTE` / `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE`; when there's no match, `url.template` is `UNMATCHED`.
  - **Never logged**: the raw path, query string, headers or bodies.
  - **Ordering**: after `RequestIdWebFilter`.

  Done when T074 and T075 pass.

**Checkpoint**: US6 complete. Observability works, with privacy enforced by a test.

---

## Phase 9: User Story 7 - The service is ready for a production environment (Priority: P3)

**Goal**: production configuration that validates safely, imports AppConfig and Secrets Manager via
Spring Cloud AWS, and a production-shaped image start in CI.

**Independent Test**: `SafeForEnvironmentTest` and `AwsConfigImportIT` pass; the CI `image` job's
production-shaped smoke passes (baseline ready; missing variable, `localhost` host and mock issuer
each exit at startup).

### Tests for User Story 7 ⚠️

- [ ] T080 [P] [US7] Write `test/config/SafeForEnvironmentTest.java` (FR-022, Story 7 #1–4; constitution §VI v1.1.0), using `ApplicationContextRunner`:
  - **`PRODUCTION` fails** when any configured address is a loopback host:
    - an R2DBC URL host `localhost`;
    - a Flyway URL host `127.0.0.1`;
    - an OTLP endpoint `http://localhost:4317`;
    - an issuer `http://[::1]`.
  - **`PRODUCTION` sidecar exception**:
    - the OTLP endpoint `http://localhost:4317` passes when its property name is listed in `budgit.sidecar-endpoints` with a reason;
    - it fails if the reason is blank.
  - **`PRODUCTION` issuer**:
    - fails with issuer `http://mock-oidc:8090/budgit`;
    - passes with `https://cognito-idp.us-east-1.amazonaws.com/us-east-1_Abc123`.
  - **`LOCAL` and `TEST`** accept both the `localhost` and the mock issuer values.
  - **Error messages** name properties and never contain their values.
- [ ] T081 [P] [US7] Write `test/config/AwsConfigImportIT.java` (research R11 first task):
  - **Setup**: start the app with the `production` profile, with WireMock stubbing the Secrets Manager `GetSecretValue` and AppConfig Data `StartConfigurationSession`/`GetLatestConfiguration` APIs. Point Spring Cloud AWS at it via `spring.cloud.aws.secretsmanager.endpoint` / `spring.cloud.aws.appconfig.endpoint` (property names per the Spring Cloud AWS 4.2 reference), and provide the remaining production values as properties.
  - **Assertions**:
    - the secret supplies the DB password;
    - an AppConfig value for `budgit.http.max-request-size` (512KB) overrides the committed 256KB;
    - without the imports configured, startup still works with env vars (`optional:`).

  First confirm the exact import prefixes in the Spring Cloud AWS 4.2 reference docs and record them in research.md R11.

### Implementation for User Story 7

- [ ] T082 [US7] Create `main/config/SafeForEnvironment.java` (a class-level constraint annotation) and `main/config/SafeForEnvironmentValidator.java` (research R10). Apply `@SafeForEnvironment` to `BudgitProperties`.
  - **What it checks**: outside `LOCAL`/`TEST`, the validator inspects the `budgit.*` URIs and, via an injected `Environment`, the Spring-owned addresses `spring.r2dbc.url`, `spring.flyway.url` and the OTLP endpoint properties.
  - **Loopback rule**: it rejects a host that is `localhost`, in `127.0.0.0/8` or `::1`, unless the property name is a key in `sidecarEndpoints` with a non-blank reason.
  - **Issuer rule**: it rejects an `issuerUri` not matching `^https://cognito-idp\.[a-z0-9-]+\.amazonaws\.com/[A-Za-z0-9-]+_[A-Za-z0-9]+$`.
  - **Errors**: messages name the property only, never its value.

  Done when T080 passes.
- [ ] T083 [US7] Create `src/main/resources/application-production.yaml` (research R11, R21):
  - **Committed values**: `budgit.environment: PRODUCTION`, and no hosts. Hosts and secrets come from env vars until the infrastructure spec decides.
  - **Imports**: `spring.config.import` with `optional:` Secrets Manager and AppConfig imports, using the prefixes confirmed in T081.
  - **Telemetry**: OTLP export enabled with endpoint `${BUDGIT_OTLP_ENDPOINT}`.
  - **Comment**: sidecar endpoints get declared under `budgit.sidecar-endpoints` when the infrastructure spec adds a collector.

  Done when T081 passes.
- [ ] T084 [US7] Create `scripts/prod-shaped-smoke.sh` and call it from the `image` job in `.github/workflows/ci.yml` (research R17, R21). It runs on a temporary Docker network with `postgres:17` (alias `db`, init scripts, `BUDGIT_ENVIRONMENT` unset) and mock-oidc (alias `mock-oidc`), and starts the image several times:
  1. **Baseline**:
     - `SPRING_PROFILES_ACTIVE=production`, `BUDGIT_DB_HOST=db`, the owner and app passwords;
     - `BUDGIT_SECURITY_ISSUER_URI=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_Smoke1`, `BUDGIT_SECURITY_JWK_SET_URI=http://mock-oidc:8090/budgit/jwks`;
     - the user and service client IDs; OTLP export disabled.

     It must reach readiness `UP` within 60 s (SC-010).
  2. **Missing variable**: the baseline without `BUDGIT_DB_APP_PASSWORD` must exit non-zero, never ready.
  3. **`localhost` host**: the baseline with `BUDGIT_DB_HOST=localhost` must exit non-zero.
  4. **Mock issuer**: the baseline with `BUDGIT_SECURITY_ISSUER_URI=http://mock-oidc:8090/budgit` must exit non-zero.

  **Also**: print any non-JSON startup lines from run 1 as a report, without failing (research R17); clean up the network and containers on exit.

**Checkpoint**: all user stories are complete.

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: verification against the spec's success criteria, documentation and repository
settings.

- [ ] T085 Create `specs/001-platform-skeleton/checklists/gate-verification.md` (SC-004) with one row per gate:
  - formatting, Error Prone, NullAway;
  - unit test, IT, JaCoCo domain threshold;
  - each ArchUnit rule (layering, ownership parameter, unscoped reason, no CrudRepository, transactional services);
  - BlockHound (or fallback);
  - Redocly lint, oasdiff (with a temporary `1.0.0` baseline);
  - dependency-check (a known-vulnerable test dependency in a throwaway branch), Trivy.

  Each row lists the throwaway change that violates it and the observed failure. Run each one and record the result; violating changes are never committed to `main`.
- [ ] T086 [P] Create `README.md` at the repository root: what the API is; prerequisites (JDK 25, Docker, `jq`, the `/etc/hosts` entry `127.0.0.1 mock-oidc`, `git config core.hooksPath .githooks`); build, run and test commands; and a link to `specs/001-platform-skeleton/quickstart.md` and the constitution.
- [ ] T087 [P] Fix the stale line in `specs/001-platform-skeleton/plan.md`'s source tree: change `test/…/config/` "AWS loader tests" to "BudgitProperties, SafeForEnvironment and AWS config import tests" (the custom loaders were replaced by Spring Cloud AWS in R11).
- [ ] T088 Configure GitHub repository settings (manual; record what was done in `specs/001-platform-skeleton/checklists/gate-verification.md`):
  - branch protection on `main`: require a pull request, and the status checks `verify`, `image` and `e2e`;
  - the `NVD_API_KEY` secret;
  - Actions workflow permissions default read-only (`publish-contract.yml` requests `contents: write` itself).
- [ ] T089 Run the full `specs/001-platform-skeleton/quickstart.md` from a fresh clone, checking SC-001 (healthy local stack in under 15 minutes with two commands) and SC-009 (PR verification in under 15 minutes in CI). Fix any step that is wrong or out of date.
- [ ] T090 Self-review the branch against the constitution v1.1.0 (constitution Workflow: "Every merge requires a self-review"). Record any deviation in plan.md Complexity Tracking, then open the PR `001-platform-skeleton` → `main`.

---

## Dependencies & Execution Order

### Phase dependencies

- **Setup (Phase 1)**: no dependencies. T008 (BlockHound spike) informs T057.
- **Foundational (Phase 2)**: depends on Setup. **Blocks all user stories.**
- **US1 (Phase 3)**: depends on Foundational.
- **US2 (Phase 4)**: depends on Foundational. T046 finishes the entry point stubbed in T026.
- **US3 (Phase 5)**: depends on Foundational (T021 contracts in place). Independent of US1 and US2.
- **US4 (Phase 6)**:
  - depends on Foundational;
  - T059–T061 also depend on US1 (`compose.yaml`, `Dockerfile`);
  - T061 also depends on T084 for the image job's smoke step (add that step when T084 lands).
- **US5 (Phase 7)**: depends on Foundational and on US2's T039–T040 (codes and exceptions) and T044 (mapping). T070 edits `ArchitectureTest.java` after T055.
- **US6 (Phase 8)**: depends on Foundational and on US2's T041 (request ID). The LogPrivacyIT routes reuse US5's test widget (T073) and US2's error probe (T036).
- **US7 (Phase 9)**: depends on Foundational. T084 depends on US1's `Dockerfile` (T030).
- **Polish (Phase 10)**: depends on all stories.

### Story dependency graph

```text
Setup ─▶ Foundational ─┬─▶ US1 ─────────────┬─▶ US4 (CI, compose.ci, e2e)
                       ├─▶ US2 ─┬─▶ US5 ─┐  │
                       │        └─▶ US6 ◀┘  │
                       ├─▶ US3              │
                       └─▶ US7 ─────────────┘ (prod-shaped smoke in US4's image job)
                                           ▼
                                        Polish
```

### Within each story

- Tests are written first and fail before implementation (constitution §IX).
- Shared types (codes, exceptions, IDs) before the components that use them.
- `application.yaml` edits are sequential: T012, T029, T047, T048, T076 and T077 touch the same
  file.
- `pom.xml` edits are sequential: T002, T005, T006, T008, T022, T029, T051, T052, T056, T057 and
  T058.
- `test/architecture/ArchitectureTest.java`: T055 → T057 (fallback only) → T070.
- `.github/workflows/ci.yml`: T061 → T084.

## Parallel Opportunities

- **Setup**: T003, T004 and T007 in parallel after T002.
- **Foundational**: T010, T011, T013, T016, T017 and T021 in parallel; then T018 → T019 → T020;
  T023 → T024; T025 → T026.
- **US1**: T027, T028, T030 and T032 in parallel.
- **US2**: tests T035–T038 in parallel; then T039 and T040 in parallel.
- **US3**: T049, T050 and T053 in parallel.
- **US4**: T058, T059, T060, T062, T063 and T064 in parallel.
- **US5**: T065, T066 and T068 in parallel; T069 in parallel with those.
- **US6**: T074 and T075 in parallel.
- **US7**: T080 and T081 in parallel.
- **Across stories** (one developer at a time is the expected case, but the work allows it): US1,
  US2, US3 and US7 can proceed in parallel after Foundational.

### Parallel example: User Story 2

```text
Write together:  T035 RequestIdIT, T036 ErrorResponsesIT, T037 CacheControlIT, T038 ServiceUnavailableIT
Then together:   T039 ProblemCode, T040 platform exceptions
Then in order:   T041 → T042 → T043 → T044 → T045 → T046 → T047 → T048
```

### Parallel example: User Story 5

```text
Together:  T065 test-schema.sql, T066 test-widgets.yaml, T068 DatabaseRolesIT, T069 typed IDs
Then:      T067 OwnershipAndVersionIT → T070 ArchUnit rules → T071 → T072 → T073
```

## Implementation Strategy

### MVP first (User Story 1)

1. Phase 1 Setup, including the T008 spike.
2. Phase 2 Foundational.
3. Phase 3 US1.
4. **Stop and validate**: quickstart §1–§3 and §8. The skeleton builds, runs locally and is healthy.

### Incremental delivery

1. **US2** (platform contract), then **US3** (contract lint and publishing). Together with US1,
   that completes the P1 stories: a runnable, contract-conformant service whose contract the UI
   can consume.
2. **US5** (ownership and versions), then **US6** (observability), then **US4** (CI workflows).
   US4's workflows can start earlier, but are most useful once there's a full test suite to run.
3. **US7** (production readiness) and its smoke test in CI.
4. **Polish**: the gate verification checklist (SC-004), README, repository settings, then a
   quickstart run from a fresh clone.

Commit after each task or logical group (`feat(001): … [Txxx]`), on `001-platform-skeleton` or on
task branches `001-platform-skeleton--Txxx-<slug>` merged back by PR (constitution Workflow).
