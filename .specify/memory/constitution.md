# BudgIT API Constitution

This constitution governs the BudgIT API (the Java service in this repository). The BudgIT UI
constitution (`budget-app/.specify/memory/constitution.md`) defines the contract the UI relies on;
this constitution governs how the API fulfills that contract. A change to any part of that shared
contract amends both constitutions.

## Core Principles

### I. Financial Correctness (NON-NEGOTIABLE)

- Money MUST be an integer in the currency's minor unit in API payloads, stored as Postgres
  `BIGINT`. Floating-point values MUST NOT be used for money anywhere.
- Each budget has a single currency, an ISO 4217 code sent to the UI as a budget setting
  (returned with the user's settings). The API
  defines the allowed currencies and sends each with its decimal shift: the number of places the
  decimal point moves to turn a stored integer into the displayed amount (e.g. 2 for USD, where
  amounts are stored in cents). Clients MUST NOT assume a shift. Whether a budget's currency can
  change is decided by a feature spec.
- In Java, money MUST be a `Money` record (`long minorUnits`, `Currency currency`) with
  overflow-checked arithmetic (`Math.addExact` and similar). Arithmetic across different
  currencies MUST fail. A raw `long` MUST NOT represent money in domain signatures. JSON sends
  amounts as integers in minor units with an explicit `currency` field.
- Amounts are signed: inflows positive, outflows negative, in the API and the database. How they
  are displayed is the UI's decision.
- Transactions and category assignments are the only source of truth. Account balances and
  Amount to Assign MUST be computed by the API from database aggregates: SQL sums transactions
  and assignments, and pure domain functions (Principle VII) apply rules such as month rollover
  to those sums.
- A cache or stored total MAY be added only for a measured need. It MUST be updated in the same
  database transaction as the write that changes it, and MUST be verifiable by recomputing it
  from transactions.
- Every write MUST return the server's values (updated records, recomputed totals, new versions)
  computed in the same database transaction as the write.

**Rationale**: A budgeting app that is off by a cent loses the user's trust. Integer minor units,
one money type and totals derived from a single source of truth remove whole classes of bugs.

### II. Non-Blocking Reactive I/O (NON-NEGOTIABLE)

- Every request path MUST be non-blocking end to end: Spring WebFlux on Netty for HTTP, R2DBC for
  Postgres, and non-blocking clients (e.g. `WebClient`, AWS SDK v2 async clients) for every
  external call. JPA/Hibernate and other blocking data access MUST NOT be used.
- The only exception is schema migrations, which run over JDBC at startup or as a deploy step
  before the app accepts traffic. That JDBC connection MUST NOT be used on a request path.
- Blocking calls MUST NOT run on Netty event-loop threads. `.block()`, `.toFuture().get()`,
  `Thread.sleep` and blocking I/O are banned in `src/main`. Unavoidable blocking work MUST be
  wrapped in `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` with a comment
  explaining why. BlockHound runs in tests to catch violations.
- Reactor is the concurrency model. Java virtual threads are not used; revisit only if a blocking
  dependency cannot be avoided.
- Pipelines MUST be composed, not nested: chain steps with `flatMap`/`then`, and use `zip` for
  independent work.
- `subscribe()` MUST NOT be called in request handling, except for deliberate fire-and-forget
  work. Such work needs a comment explaining why, handles its own errors, runs on
  `boundedElastic` if it blocks, and is never work that must happen (financial changes, or
  anything in the write's transaction).
- Operators MUST NOT share mutable state. Per-request data (user, request ID) travels in the
  Reactor `Context` (or Spring features built on it), never in a `ThreadLocal`.
- Every external call MUST have a timeout.

**Rationale**: One blocking call on an event-loop thread stalls every request on that thread. A
strict, testable rule keeps the API fast under load.

### III. Dates and Time

- Calendar days (e.g. transaction dates) are `YYYY-MM-DD` with no timezone in the API, stored as
  Postgres `DATE`, and MUST NOT be converted.
- Moments (`created_at`, `updated_at`, import time) are ISO 8601 UTC strings in the API and stored
  as `timestamptz` in UTC.
- A budget month is a single `YYYY-MM` value.
- Bank and CSV timestamps become a calendar day once, at import, using the timezone in the user's
  profile settings, never the browser's.
- Java types: `LocalDate` for calendar days, `YearMonth` for budget months, `Instant` for moments,
  and `ZoneId` from the user's settings.
- The current time MUST come from an injected `Clock`. `Instant.now()`, `LocalDate.now()` and
  similar MUST NOT be called without it.

**Rationale**: Every viewer must see a transaction on the same day and in the same budget month.
Converting calendar days, or depending on the server's clock and zone, breaks that silently.

### IV. Contract-First API

- The API is contract-first. An OpenAPI 3.1 spec in `src/main/resources/openapi/` is the source of
  truth. `openapi-generator` (`spring`, `reactive=true`, `interfaceOnly=true`) generates controller
  interfaces and DTOs. Controllers are annotated `@RestController` classes returning
  `Mono`/`Flux` that implement the generated interfaces.
- The spec MUST be published as a versioned artifact (e.g. a GitHub release asset) on every merge
  to `main`. The UI pins a version and generates its API types and MSW mock types from it.
- CI MUST fail on breaking spec changes (e.g. `oasdiff`) unless the major API version changes.
  This applies from the first stable contract version (`1.0.0`, released with the MVP). Before
  that, the contract is versioned `0.x`, carries no stability promise (as in semantic
  versioning), and breaking changes do not block a merge.
- Integration tests MUST validate every request and response against the spec (e.g.
  `swagger-request-validator`), so the API cannot drift from the UI's mocks.
- URLs: `/api/v1/...`, plural resource nouns, kebab-case paths, camelCase JSON fields, and budget
  months in paths as `YYYY-MM` (e.g. `/api/v1/budgets/{budgetId}/months/2026-09`).
- Errors MUST use RFC 9457 Problem Details (`application/problem+json`) with a stable `code`
  (e.g. `STALE_VERSION`, `VALIDATION_FAILED`), per-field errors and the request ID. Error bodies
  MUST NOT contain financial data or internals.
- Status codes:
    - 200 for reads and updates (with the resulting state), 201 for creates.
    - 400 for validation errors, 401 when unauthenticated, 404 for missing records and for other
      users' records (so IDs cannot be probed), 409 for stale versions and create conflicts, 413
      for uploads over the size limit, 415 for unsupported content types, 422 for business-rule
      violations, 429 for rate limits.
    - 500 for unexpected errors, with a generic body.
    - 503 when a dependency such as the database is temporarily unavailable or the API is shutting
      down, with `Retry-After` when known.
    - 502 or 504 when an external service fails or times out.
- Reads MAY be retried automatically only on server errors (5xx). Reads MUST NOT be retried on
  client errors (4xx), which a retry cannot fix. Callers MUST NOT retry writes automatically,
  unless the endpoint is idempotent by design (e.g. user provisioning). The UI constitution applies
  this rule to the browser and the Next.js server.
- Input shape (required fields, lengths, `YYYY-MM-DD` and `YYYY-MM` formats, and so on) MUST be
  validated with Jakarta Bean Validation on the generated DTOs. Business rules are validated in
  the domain layer. Unknown JSON fields MUST be rejected.
- Every write response MUST contain the updated records, recomputed totals and new versions, so
  the UI never reads again to see its own write.
- Reads MUST be cheap and safe to repeat, because the UI refetches on window focus and reconnect.
  Every response MUST carry `Cache-Control: no-store`.

**Rationale**: The UI and its mock API are built from the spec before endpoints exist. Keeping the
spec as the source of truth, and testing against it, keeps both sides in step.

### V. Persistence and Data Integrity

- The database is PostgreSQL on AWS: Amazon RDS for PostgreSQL to start, with the option to move
  to Aurora PostgreSQL. The API MUST use only standard PostgreSQL features available on both, stay
  on a major version both support, and keep connection details in config, so a move needs no code
  changes. Local development and CI use the same major version in Docker.
- All request-path database access runs over R2DBC with `r2dbc-pool`. Spring Data R2DBC
  repositories are used for simple single-table CRUD. Aggregate queries (e.g. account balances
  and the per-category, per-month sums behind Amount to Assign) and performance-sensitive queries
  use hand-written SQL through `DatabaseClient`, or `@Query` for simple cases. jOOQ over R2DBC is
  an allowed alternative if type-checked SQL becomes worth it.
- Transactions are declared with `@Transactional` on public service methods, or
  `TransactionalOperator` when only part of a pipeline should be transactional. Each user action
  MUST be one database transaction covering the change and the totals returned in the response.
- Transactions MUST stay short: no external calls inside them and no work escaping the pipeline
  through `subscribe()`. Multi-query reads that must be mutually consistent use a read-only
  transaction.
- If read replicas are ever added, reads that follow a write MUST go to the primary.
- Schema changes use Flyway over a JDBC connection used only for migrations. Migrations are
  forward-only and MUST be compatible with the previous app version (expand now, contract in a
  later release).
- Every record has a UUID primary key, exposed as a string. The UUID version is not fixed.
- The client MAY send a UUID on create, so the UI can create records optimistically. Repeating a
  create with the same ID MUST return the existing record rather than a duplicate; an ID that
  already belongs to a different record returns 409. The server generates an ID if none is sent.
- Every mutable table has an integer `version`, checked and incremented in the
  `UPDATE ... WHERE version = ?`. A mismatch MUST return 409 with code `STALE_VERSION`; the last
  save never silently wins. Because of this check, updates are safe to retry. `Idempotency-Key`
  headers are deferred until a feature needs them (e.g. bank imports).
- Deletes are hard deletes unless a feature spec needs a restore window. Any soft-deleted rows
  MUST be hard-deleted by a scheduled job when that window ends. Deletes that would leave other
  records pointing at a removed record (e.g. a category with transactions) MUST resolve those
  references at delete time.

**Rationale**: Version checks, single-transaction writes and backward-compatible migrations keep
financial data consistent through concurrent edits, retries and deploys.

### VI. Security and Privacy

- Every caller of the API is one of the kinds below. Adding a new caller requires an explicit
  decision recorded in a feature spec.
    - **User-facing callers** act for a signed-in user and send that user's Cognito access token
      as `Authorization: Bearer <access token>`. The Next.js server (backend-for-frontend) is
      currently the only one. The API takes the user's identity only from the token, never from a
      request parameter.
    - **Service callers** (e.g. the user-provisioning function, scheduled jobs) authenticate with
      Cognito client-credentials tokens: one app client per caller, with a scope per task. They
      call only internal endpoints that require their scope. Internal endpoints are not publicly
      reachable and are described in a separate internal OpenAPI spec, not in the contract
      published to the UI. A service token MUST NOT be accepted on user endpoints, and a user token
      MUST NOT be accepted on internal endpoints.
    - **Third parties** that cannot obtain Cognito tokens (e.g. bank-linking webhooks)
      authenticate with the provider's own mechanism (e.g. signed requests), through a dedicated
      entry point decided in that feature's spec.
- The API is not reachable from the browser, so it handles no cookies, CSRF or CORS. CSRF
  protection is enforced by Next.js, which also maps the API's 401 to the browser redirect to
  `/login`, except a 401 with code `USER_NOT_PROVISIONED` (sign-up not yet finished), which the UI
  shows as a "finishing setup, try again" message. Next.js MUST check that code before refreshing
  tokens or redirecting, so it never loops on it.
- Sessions are owned by the Next.js server (UI constitution §VI). The API never sees the session
  ID; it receives only the bearer token.
- The API is a Spring Security WebFlux OAuth2 resource server. It MUST validate Cognito JWT access
  tokens (issuer, client ID, expiry, `token_use=access`, scopes for service callers, and other
  claims as needed) against the cached JWKS.
- The backend enforces authorization on every request. Every query MUST be scoped by the owning
  user (or budget) in the repository layer, enforced by ArchUnit and integration tests. A record
  belonging to another user returns 404.
- The schema MUST be ready for Postgres row-level security as defense in depth: every user-owned
  table carries its own `user_id`; the API connects as a non-owner, non-superuser role, with
  migrations using a separate owner role; and all database work for a request runs in a
  transaction so a per-transaction user setting can be added later.
- Users are provisioned by an event from the identity provider's sign-up confirmation, not on
  first request. Provisioning MUST be idempotent. Each user has an internal ID generated by the
  API, which every other table references. The identity provider's ID (Cognito `sub`) is stored
  only on the user record, so changing user pools or providers touches no other data. Profile
  data (email, name) is not copied unless a feature needs it.
- Every budget has its own unique ID, so more than one budget per user can be added later.
- A user MUST be able to delete their account and all their data, including soft-deleted rows, as
  a hard delete, before the app is offered to other people.
- Amounts, balances, payees, memos, account names or numbers, tokens, session IDs and emails MUST
  NOT appear in logs, metrics, traces, analytics, error reports or URLs. Record IDs, error codes and
  request IDs are allowed. Request and response bodies MUST NOT be logged.
- Any new third party that receives financial data requires its own explicit decision.
- Config follows the same rules as the UI: committed per-environment config files uploaded by CI
  and read at startup, overridden by AWS AppConfig. Secrets come from environment variables or AWS
  Secrets Manager and MUST NOT be committed. Config is loaded into `@Validated`
  `@ConfigurationProperties` classes, and the API MUST refuse to start if a value is missing or
  points at `localhost` outside the `local` and `test` environments. The one exception is a
  sidecar: a helper container deployed next to the API and reached on `localhost` (e.g. a
  telemetry collector). A sidecar endpoint MUST be declared as such in the committed
  config, naming the setting and the reason. Any other `localhost` value still stops startup.
- Startup validation MUST refuse a configured trusted token issuer that is not a Cognito issuer
  in every environment except `local` and `test`, where a mock OIDC server stands in for Cognito
  (Principle IX). Tokens from the mock server are still real, signed tokens validated the same
  way; there is no test-only auth bypass.
- Dependencies MUST be scanned for known vulnerabilities in CI on every pull request and on a
  schedule against `main`, failing on high or critical findings. The deployed Docker image is
  scanned too. Automated update PRs (e.g. Dependabot) keep dependencies current. Suppressing a
  finding requires a comment explaining why and a date to re-check.

**Rationale**: The API holds personal financial data. Isolating every query by owner, keeping
secrets and financial values out of logs, and having a single trusted caller limit the damage of
any one mistake.

### VII. Domain Logic and Code Organization

- Business rules (money math, Amount to Assign, target progress, month rollover, split validation)
  MUST be pure Java functions over records and sealed interfaces, with no Spring, Reactor or I/O.
  Services do the reactive I/O and call them.
- Collections in domain code SHOULD be processed with streams (`filter`, `map`, `reduce`,
  `collect`) rather than imperative loops, with no side effects inside stream steps.
- The API is the authority on business rules (validation, splits, date bucketing). Rules checked
  by both the API and the UI MUST share one set of test fixtures (JSON inputs and expected
  results) run against both implementations. The fixtures live in
  `src/test/resources/shared-fixtures/` and are published with the OpenAPI spec; the UI pulls a
  pinned version.
- Packages are organized by layer: `api` (controllers and API mappers), `service` (orchestration
  and transactions), `domain` (pure business rules), `persistence` (repositories, SQL and row
  mappers), `config` and `common`, with generated OpenAPI code in its own package. Classes MAY be
  grouped by feature within a layer. Moving to another structure (by feature, multi-module) is
  allowed if the codebase outgrows this.
- ArchUnit MUST enforce the layering: `domain` does not depend on Spring, Reactor or persistence;
  controllers do not call repositories; repository query methods take an owner or budget ID.
- Generated OpenAPI DTOs MUST NOT reach the domain layer. Controllers map DTOs to domain records
  and back.
- DTOs and domain values are Java records with small hand-written mappers. Lombok and MapStruct
  are not used.
- Packages are JSpecify `@NullMarked`, checked by NullAway. `Optional` is used only as a return
  type. In reactive code, `Mono.empty()` means "not found".
- A class that takes on a second responsibility MUST be split. Classes SHOULD stay under 300–400
  lines; this is a heuristic, not a hard limit.

**Rationale**: Pure domain functions are easy to test, easy to run against shared fixtures, and
keep reactive plumbing out of the rules that decide where money goes.

### VIII. Budget Domain Rules

- Amount to Assign: inflows categorized as "Ready to Assign" add to it; assignments in any month
  subtract from it; a category's available amount rolls into the next month; cash overspending in
  a category reduces the next month's Amount to Assign.
- A transaction without a category counts as Ready to Assign until the user picks a category: an
  uncategorized outflow reduces Amount to Assign and an uncategorized inflow adds to it.
- Amount to Assign MAY go negative. The API accepts the writes that cause it and returns the
  negative value as is; warning the user is the UI's responsibility.
- Account types are Cash and Credit. Spending on a credit card moves money from the spending
  category to an automatic payment category for that card.
- A transfer is a linked pair of transactions with no category. A split is one transaction whose
  category lines MUST sum exactly to its total.
- Target progress ("needed this month", "underfunded") MUST be computed in the API, returned with
  the budget month, as pure domain functions covered by the shared fixtures.
- Category and line-item order is stored as an explicit sort key per row (gap-based integers or
  fractional keys), so moving a row changes one row.
- User preferences that follow the user across devices, including the profile timezone, are user
  settings saved through the API.

**Rationale**: These rules decide where every dollar goes. Fixing them at the constitution level
keeps the API, the UI and the shared fixtures in agreement.

### IX. Test-First Domain Logic

- Test-first development is REQUIRED for domain logic (money, balances, Amount to Assign, targets,
  imports) and for each endpoint's contract behavior.
- Tools: JUnit 6 and AssertJ for unit tests, Reactor `StepVerifier` for reactive flows,
  `WebTestClient` for endpoint tests, Testcontainers Postgres for repository and integration tests,
  and BlockHound in integration tests. H2 and mocked databases MUST NOT be used.
- `domain` packages MUST keep at least 85% coverage, enforced with JaCoCo in CI. There is no
  minimum elsewhere.
- Integration tests against a real database MUST cover at least: every write (including its
  returned totals), stale-version rejection, cross-user access returning 404, and the page
  object returned by list endpoints (real paging once it is introduced).
- Sample data is used only in tests, a dev-only seed and a staging environment if one exists. It
  MUST NOT be present in production.
- A Docker Compose stack (Postgres, the API, the Next.js UI, and a mock OIDC server standing in for
  Cognito) MUST be startable by both repositories' CI. API CI runs it with the new API and the
  latest released UI; UI CI runs it with the new UI and the latest released API. Sign-in goes
  through the real OIDC flow against the mock server. The UI's smaller end-to-end suite runs
  against this stack.

**Rationale**: Money bugs are silent and costly. Pure, tested domain functions and integration
tests against real Postgres catch them before users do.

### X. Performance and Resilience

- The budget loads one month at a time; totals are computed by the API. Transaction lists are returned as
  page objects, and a single page contains everything until paging is introduced. Whether and
  when to page, its form (e.g. cursor or date range) and the threshold that triggers it are
  decided later by a feature spec.
- Latency targets (p95): under 150 ms for a budget-month read, under 100 ms for a transactions
  page, and under 150 ms for a write including recomputed totals. The data volume they are
  measured against is set by the spec that introduces the measurement.
- Each query MUST be designed with its indexes and checked with `EXPLAIN` in review. Sums
  MUST be computed in SQL. Individual transactions or assignments MUST NOT be loaded into Java to
  total them; only the aggregated results are. Cached totals are allowed only as in
  Principle I.
- Every outbound call has a timeout. Retries, with backoff and jitter, are allowed only for
  idempotent calls.
- CSV imports are uploaded to the API and parsed as a stream, with a size limit (e.g. 5 MB / 10k
  rows), duplicate detection by a per-row fingerprint, and the whole import written in one
  database transaction.
- Netty and connection-pool settings start at defaults and MAY be tuned when performance tests
  show a problem.

**Rationale**: Measurable targets keep the API fast as data grows without premature optimization.

### XI. Observability and Operations

- The application writes structured JSON logs (Spring Boot structured logging) to standard output
  only and never ships logs itself. The platform collects and forwards them (CloudWatch Logs to
  start with), so the destination can change without code changes.
- Every log line carries the request ID, internal user ID and trace ID, propagated through Reactor
  with Micrometer context propagation.
- A test MUST check log output for the privacy rules in Principle VI.
- Logs are kept for a limited period (e.g. 30 days), encrypted, and readable only by the owner and
  the deployment pipeline.
- Metrics use Micrometer and traces use OpenTelemetry, exported to CloudWatch or another chosen
  backend. Financial values MUST NOT appear in metric tags or span attributes.
- The API accepts `X-Request-Id` from the Next.js server or generates one, and returns it on every
  response and in every Problem Details body.
- Only the `health` (liveness and readiness) and `info` Actuator endpoints are exposed, on a
  separate management port that is not publicly reachable.

**Rationale**: Request IDs and structured logs make problems traceable without ever recording the
financial data the app exists to protect.

## Scope, Relationship to the UI and Technology Constraints

- **Product**: the API for BudgIT, a personal, zero-based budgeting web app for one user per
  budget, covering monthly budgeting, targets, accounts and transactions.
- **Not supported**: shared or multi-user budgets, investments, more than one currency in a
  budget, currency conversion, and server push of changes (Server-Sent Events or WebSockets).
  Adding any of these requires amending this constitution and the UI constitution.
- **Feature order**: which features are built, and when, is decided by feature specs and the
  backlog, not by this constitution.
- **Relationship to the UI constitution**: the UI constitution defines the contract the UI relies
  on; this constitution governs how the API fulfills it. A change to that contract (money in minor
  units and currency, computed totals, date formats, write responses, versions, generated types,
  authorization, privacy, sample data, workflow and governance) amends both.
- **Stack**: Java 25, Spring Boot 4.x (latest GA, Spring Framework 7), Spring WebFlux on Netty,
  Project Reactor, Spring Security (OAuth2 resource server), R2DBC with `r2dbc-pool`, PostgreSQL,
  Flyway, and Amazon Cognito (OIDC) for identity. Spring Boot minor versions are upgraded within a
  month of release.
- **Build**: Maven, with the Maven Wrapper committed and `spring-boot-starter-parent` as the
  parent.
- **Deployment**: a Docker image on an AWS container orchestration service (ECS or EKS; to be
  decided). Local development uses Docker Compose, which is also the end-to-end test environment.
- **Environments**: `local`, `test` (CI) and `production`, matching the UI's per-environment config
  files. More environments MAY be added later.

## Development Workflow and Quality Gates

- Features MUST have a Spec Kit spec before any code is written. Bug fixes and chores skip specs.
- Features that span both repositories get one Spec Kit spec per repository, each with its own
  feature number. The API spec and its OpenAPI change land first; the UI spec references the API
  repository and feature number.
- A merge MUST be blocked unless `./mvnw verify` passes in GitHub Actions: unit and integration
  tests (Testcontainers), the JaCoCo threshold, ArchUnit, a Spotless check, Error Prone with
  NullAway, OpenAPI lint and breaking-change check, and the dependency scan.
- Spotless with Palantir Java Format formats all code, applied by a pre-commit hook.
- `@SuppressWarnings` and Error Prone or NullAway suppressions are allowed only with a comment
  explaining why, in every package including `domain`.
- Commits follow Conventional Commits and reference the Spec Kit feature number and task ID, e.g.
  `feat(003): add split validation [T004]`.
- One branch per feature, named by Spec Kit (e.g. `003-split-transactions`), merged into `main` by
  pull request when the spec is complete. Task branches (e.g.
  `003-split-transactions--T004-validation`) may branch off the feature branch and merge back by
  pull request. Spec Kit commands run on the feature branch, which pulls in `main` regularly.
- Every merge requires a self-review against this constitution.

## Governance

- This constitution supersedes other API practices. Where existing code conflicts with it, new and
  changed code MUST follow the constitution; existing code migrates as it is touched or through
  planned migration tasks.
- Only the project owner may amend it. Amendments go through a pull request that includes a version
  bump and a note on how existing code moves to the new rule. An amendment that changes the
  contract shared with the UI MUST also amend the UI constitution.
- Versioning: MAJOR when a principle is removed or reversed; MINOR when a principle or section is
  added or materially expanded; PATCH for wording-only changes.
- Every plan's Constitution Check and every pre-merge self-review MUST verify compliance with these
  principles. Any deviation MUST be justified in the plan's Complexity Tracking table.

**Version**: 1.1.0 | **Ratified**: 2026-10-12 | **Last Amended**: 2026-10-18
