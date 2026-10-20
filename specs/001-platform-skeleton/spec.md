# Feature Specification: Platform Skeleton

**Feature Branch**: `001-platform-skeleton`

**Created**: 2026-10-12

**Status**: Draft

**Input**: User description: "Read mvp-requirements.md - create the spec for the platform skeleton PLT-1 to PLT-10"

**Source requirements**: `docs/mvp-requirements.md` §2.7 (PLT-1 to PLT-10) and §6 item 1; API
constitution v1.1.0 (cited as "API §N").

## Overview

The platform skeleton is the first BudgIT API feature. It delivers no budgeting functionality.
Instead it delivers a running, deployable, observable service with an empty published contract,
plus every shared mechanism and quality gate the constitution requires of later features, so that
the next spec (Users and auth) can add endpoints without building any platform pieces itself.

The "users" of this feature are:

- **The owner/developer**, who builds and operates the API.
- **The UI repository** (`budget-app`), which consumes the published contract, shared fixtures
  and the end-to-end stack.
- **Operators of the production environment** (currently the same person as the owner).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Run the service locally and see it healthy (Priority: P1)

The developer clones the repository, builds it with the committed build wrapper, starts the local
stack with one command, and sees the API report itself healthy and ready, with its database schema
migrated. The service rejects unauthenticated calls to the public API with a well-formed error.

**Why this priority**: Nothing else (contract, CI, deployment, later features) can be built or
verified without a runnable service and a local environment.

**Independent Test**: On a clean machine with only the documented prerequisites installed, build
the project, start the local stack, and query the health and info endpoints on the management
port.

**Acceptance Scenarios**:

1. **Given** a fresh clone and the documented prerequisites, **When** the developer runs the build
   command, **Then** the build compiles, runs all tests and gates, and succeeds without any
   globally installed build tool.
2. **Given** the build has succeeded, **When** the developer starts the local stack with a single
   command, **Then** the database, the API, the UI (latest released version) and the mock identity
   provider start, and the API reports "ready" on its management port once its schema migrations
   have run.
3. **Given** the local stack is running, **When** the developer requests the health and info
   endpoints on the management port, **Then** liveness, readiness and build info are returned, and
   no other operational endpoints are exposed.
4. **Given** the local stack is running, **When** any request is sent to a public API path on the
   application port, **Then** the health and info endpoints are not reachable there.
5. **Given** the local stack is running, **When** the developer loads the dev-only seed, **Then**
   sample data is available locally, and the seed cannot be loaded in any environment other than
   `local`.

---

### User Story 2 - Every response follows the platform contract (Priority: P1)

Any caller of the API (initially the Next.js server and the mock identity provider's test tokens)
gets consistent behavior from every endpoint: authentication is required, errors are Problem
Details with a stable code and the request ID, every response is marked non-cacheable, and every
request is traceable by its request ID.

**Why this priority**: These behaviors are constitution gates for every endpoint. Building them
once, before any business endpoint exists, keeps later features from re-implementing or
forgetting them.

**Independent Test**: Against the running service, send requests with and without a valid token,
with and without an `X-Request-Id` header, to an unknown path, and with a malformed body, and
check status codes, headers and error bodies.

**Acceptance Scenarios**:

1. **Given** a request to a public API path with no bearer token, an expired token, a token from
   an untrusted issuer, or an ID token instead of an access token, **When** the API handles it,
   **Then** it returns 401 as Problem Details with a stable code, and no body data beyond the error.
2. **Given** a request with an `X-Request-Id` header, **When** the API responds (success or error),
   **Then** the same request ID is returned in the response header, appears in every log line for
   that request, and is included in any Problem Details body.
3. **Given** a request with no `X-Request-Id`, **When** the API responds, **Then** it generates a
   request ID and returns it the same way.
4. **Given** any request, **When** the API responds with any status, **Then** the response carries
   `Cache-Control: no-store`.
5. **Given** a request to a path that does not exist, **When** the API handles it, **Then** it
   returns 404 Problem Details with a stable code.
6. **Given** an unexpected failure inside the service, **When** the API handles it, **Then** it
   returns 500 with a generic Problem Details body containing no internals, stack traces or
   financial data.
7. **Given** the database is unavailable, or the service is shutting down, **When** a request
   arrives, **Then** it returns 503 Problem Details, with `Retry-After` when known, and readiness
   reports "not ready".
8. **Given** a request with an unsupported content type, or a JSON body with unknown fields,
   **When** the API handles it, **Then** it returns 415 or 400 respectively, as Problem Details.
9. **Given** a service-caller token (client-credentials), **When** it is sent to a public user
   endpoint, **Then** it is rejected; **and given** a user token sent to an internal endpoint,
   **Then** it is rejected.

---

### User Story 3 - The contract and shared fixtures are published for the UI (Priority: P1)

The API's public contract (OpenAPI 3.1, empty of business endpoints at this stage) and the shared
test fixtures are published as a versioned artifact on every merge to `main`, so the UI repository
can pin a version and generate its types and mocks from it. The internal contract (service-caller
endpoints) is kept separate and is not published to the UI.

**Why this priority**: The constitution's workflow requires each API spec's contract change to
land before the matching UI spec. The publishing pipeline must exist before the first endpoint.

**Independent Test**: Merge a change to `main` and verify a new versioned artifact containing the
public contract and the shared fixtures appears, and that a consumer can download a specific
version.

**Acceptance Scenarios**:

1. **Given** a merge to `main`, **When** the pipeline completes, **Then** a new, uniquely versioned
   artifact containing the public contract and the shared fixtures is published and downloadable
   by version.
2. **Given** the published artifact, **When** its contents are inspected, **Then** it contains no
   internal (service-caller) endpoints.
3. **Given** a contract at version `1.0.0` or later, **When** a pull request makes a breaking
   change without a major version change, **Then** the merge is blocked; **and given** a
   contract below `1.0.0`, **Then** the breaking change is reported but does not block.
4. **Given** a pull request whose contract fails linting, **When** the gates run, **Then** the
   merge is blocked.
5. **Given** an integration test sends a request or receives a response that does not match the
   contract, **When** the test suite runs, **Then** the test fails.

---

### User Story 4 - Merges are blocked unless every constitution gate passes (Priority: P2)

Every pull request runs the single build-and-verify command in CI, which enforces every gate the
constitution lists. A scheduled run checks `main` for newly disclosed dependency vulnerabilities.

**Why this priority**: The gates must exist before features land, but a locally runnable service
(P1) is needed to exercise them.

**Independent Test**: Open pull requests that each deliberately violate one gate and confirm each
is blocked; open a clean pull request and confirm it passes.

**Acceptance Scenarios**:

1. **Given** a pull request, **When** CI runs, **Then** it runs unit tests, integration tests
   against a real database in a container, the domain coverage threshold, the architecture
   (layering) rules, formatting check, static analysis with null-safety checking, contract lint and
   breaking-change check, and the dependency vulnerability scan, and fails if any one fails.
2. **Given** a pull request whose code calls a blocking operation on a request path, **When**
   integration tests run, **Then** the blocking call is detected and the build fails.
3. **Given** a pull request where the domain layer depends on the web framework, persistence or
   the reactive library, or a controller calls a repository directly, or a repository query method
   lacks an owner or budget ID parameter, **When** the architecture rules run, **Then** the build
   fails.
4. **Given** a dependency or the built container image has a known high or critical
   vulnerability with no documented, dated suppression, **When** the scan runs, **Then** the build
   fails.
5. **Given** the schedule fires, **When** the scan runs against `main`, **Then** new high or
   critical vulnerabilities fail the scheduled run visibly.
6. **Given** CI runs, **When** the end-to-end job runs, **Then** it starts the same Compose stack
   used locally, with the new API and the latest released UI.

---

### User Story 5 - Shared mechanisms for ownership and concurrency are ready (Priority: P2)

Later features will store user-owned, mutable records. The skeleton provides the shared mechanisms
they will reuse: owner-scoped data access where another user's record behaves exactly like a
missing one (404), optimistic version checks returning `409 STALE_VERSION`, and a schema and
database-role setup ready for row-level security.

**Why this priority**: These are constitution gates (API §V, §VI) for every future mutable record.
They have no business endpoint to attach to yet, so they are proven with test-only records.

**Independent Test**: Using a test-only owned, versioned record, verify that updating with a stale
version yields 409 `STALE_VERSION`, that reading or updating another owner's record yields 404, and
that the application's database role cannot alter the schema.

**Acceptance Scenarios**:

1. **Given** a test-only record owned by user A at version 3, **When** an update is sent with
   version 2, **Then** the update is rejected with 409 Problem Details code `STALE_VERSION` and the
   record is unchanged.
2. **Given** the same record, **When** an update is sent with version 3, **Then** it succeeds and
   the response carries version 4.
3. **Given** a record owned by user A, **When** user B reads, updates or deletes it, **Then** the
   response is 404, indistinguishable from a record that does not exist.
4. **Given** the running service, **When** its database connection role is inspected, **Then** it
   is not the schema owner and not a superuser, and migrations run under a separate owner role.
5. **Given** a create with a client-supplied ID that already exists for the same record, **When**
   it is repeated, **Then** the existing record is returned; **and given** the ID belongs to a
   different record, **Then** 409 is returned.

---

### User Story 6 - The service is observable without leaking financial data (Priority: P2)

The operator can follow any request through logs, metrics and traces by its request ID, user ID
and trace ID, and can be sure no financial or personal data ever appears in that telemetry.

**Why this priority**: Required by API §XI before production traffic; needed to operate the
production environment (Story 7).

**Independent Test**: Send requests containing recognizable sample amounts, names, memos and tokens
and assert that captured log output, metric tags and trace attributes contain none of them, while
containing the request ID and trace ID.

**Acceptance Scenarios**:

1. **Given** any request, **When** it is handled, **Then** every log line written while handling
   it is a single structured JSON object on standard output carrying that request's ID and trace
   ID (and the internal user ID once users exist).
2. **Given** a request whose headers or body contain amounts, account names, payees, memos, tokens
   or emails, **When** the log privacy test captures all output for that request, **Then** none of
   those values appear, and no request or response bodies are logged.
3. **Given** the service is running, **When** metrics and traces are exported, **Then** request
   metrics and spans exist for each request and contain no financial values in tags or attributes.

---

### User Story 7 - The service is ready for a production environment (Priority: P3)

The API runs in a single production environment on AWS with a managed PostgreSQL database, reads
its configuration from committed per-environment files overridden at runtime by remote
configuration, reads secrets from a secret store, and refuses to start if its configuration is
unsafe.

**Why this priority**: Required for the MVP but only useful once there are features to serve; the
local and test environments are enough to build the next specs.

**Independent Test**: Start the service with missing or unsafe configuration in a non-local
environment and confirm it refuses to start; start the image with valid production-shaped configuration and
confirm health is green.

**Acceptance Scenarios**:

1. **Given** any environment, **When** a required configuration value is missing or invalid,
   **Then** the service refuses to start and names the offending setting (never its secret value).
2. **Given** any environment other than `local` and `test`, **When** a configured address points
   at `localhost` and that setting is not declared as a sidecar endpoint, **Then** the service
   refuses to start; **and given** the setting is declared as a sidecar endpoint with a reason,
   **Then** the service starts.
3. **Given** any environment other than `local` and `test`, **When** the configured trusted token
   issuer is not a Cognito issuer (e.g. it points at the mock OIDC server), **Then** the service
   refuses to start.
4. **Given** `local` or `test`, **When** the configured trusted token issuer is the mock OIDC
   server, **Then** the service starts and validates the mock server's tokens the same way it
   would validate Cognito's.
5. **Given** production, **When** remote configuration overrides a committed value, **Then** the
   service uses the overridden value.
6. **Given** production, **When** the service starts, **Then** secrets come only from the secret
   store or environment variables, never from committed files.
7. **Given** the container image and the committed `production` configuration, **When** the image
   is started with production-shaped settings (Cognito issuer, non-local database address, secrets
   from environment variables), **Then** it passes startup validation and reports ready. Setting up
   AWS infrastructure and deploying to it are out of scope (see Assumptions).

---

### Edge Cases

- A request carries an `X-Request-Id` that is empty, too long or contains unsafe characters: the
  API replaces it with a generated ID rather than echoing it.
- The database is not yet reachable at startup: the service waits/retries migrations within a
  bounded time, reports "not ready" meanwhile, and fails startup if the bound is exceeded.
- A migration fails at startup: the service does not start serving traffic.
- The identity provider's key set cannot be fetched: token validation fails closed (401 or 503),
  never open, and the fetch has a timeout.
- A token is valid but signed with a key that has rotated: the cached key set is refreshed once
  before rejecting.
- The service receives a shutdown signal mid-request: in-flight requests complete within a grace
  period; new requests get 503 and readiness turns "not ready".
- An error message would include a field value from the request: only the field name and error
  code are returned, never the value.
- No released UI version exists yet: the end-to-end stack runs and the API-side checks pass, with
  the UI service omitted until a release exists.
- The dev-only seed is invoked in any environment other than `local`: it refuses to run.
- A Problem Details response for 401/404 must not reveal whether a resource exists or which check
  failed beyond the stable code.

## Requirements *(mandatory)*

### Functional Requirements

**Runnable service and structure (PLT-1)**

- **FR-001**: The repository MUST build, test and verify with a single committed wrapper command,
  requiring no globally installed build tool beyond the documented runtime prerequisites.
- **FR-002**: The service MUST use the stack mandated by the constitution (Stack, API §II, §V),
  with database schema managed by forward-only, versioned migrations that run before the service
  accepts traffic.
- **FR-003**: The codebase MUST be organized into the layers named in API §VII, and automated
  architecture rules MUST fail the build on any layering violation listed there.
- **FR-004**: An automated check MUST fail the build if a blocking call occurs on a request path
  during integration tests.

**Contract-first API (PLT-2)**

- **FR-005**: A public OpenAPI 3.1 contract MUST exist as the source of truth, containing the
  shared Problem Details error schema and security scheme and no business endpoints yet.
- **FR-006**: A separate internal contract MUST exist for service-caller endpoints and MUST NOT be
  included in the published artifact.
- **FR-007**: Server interfaces and request/response types MUST be generated from the contracts as
  part of the build.
- **FR-008**: Integration tests MUST validate every request and response they exchange against the
  contract and fail on any mismatch.
- **FR-009**: On every merge to `main`, the public contract and the shared fixtures directory MUST
  be published as one uniquely versioned, downloadable artifact.
- **FR-010**: The build MUST lint the contract. From contract version `1.0.0` onward, it MUST
  fail on breaking changes to the public contract unless the major version changes. Below
  `1.0.0`, breaking changes MUST be reported but MUST NOT fail the build (API §IV, constitution
  v1.1.0).

**Errors and HTTP conventions (PLT-3)**

- **FR-011**: Every error response MUST be RFC 9457 Problem Details with a stable machine-readable
  `code`, per-field errors where applicable, and the request ID, and MUST NOT contain financial
  data, request values or internals.
- **FR-012**: The service MUST use the status codes listed in API §IV for the conditions handled at
  platform level: 400 (validation, unknown JSON fields), 401, 404 (including unknown paths), 409,
  413, 415, 500 (generic body), 503 (with `Retry-After` when known), and 502/504 for failing or
  timed-out external calls.
- **FR-013**: Every response, success or error, MUST carry `Cache-Control: no-store`.
- **FR-014**: The service MUST accept a well-formed `X-Request-Id` from the caller or generate one,
  return it on every response and in every Problem Details body, and make it available to all
  logging for that request.

**Concurrency (PLT-4)**

- **FR-015**: The platform MUST provide a reusable optimistic version mechanism such that an update
  to a mutable record with a mismatched version is rejected with 409 code `STALE_VERSION` and
  leaves the record unchanged, and a successful update increments the version.
- **FR-016**: The platform MUST provide reusable create-with-client-ID handling:
  - a create that inserts a new record returns `201 Created`;
  - repeating a create with an ID that already exists for the same owner returns
    `200 OK` with the existing record, unchanged;
  - an ID belonging to a different record returns `409 ID_CONFLICT`.

  Every create endpoint in later specs follows these status codes.

**Ownership and security (PLT-5, auth foundation)**

- **FR-017**: The platform MUST provide owner-scoped data access such that every repository query
  requires an owner (user or budget) ID, enforced by architecture rules, and a record belonging to
  another owner is indistinguishable from a missing one (404).
- **FR-018**: The schema conventions MUST be row-level-security ready: every user-owned table
  carries its own user ID; the service connects as a non-owner, non-superuser role; migrations use
  a separate owner role; and all request database work runs inside a transaction.
- **FR-019**: Every public API path MUST require a valid identity-provider access token (issuer,
  audience/client, expiry, access-token type, and signature against a cached key set). Service
  tokens MUST be rejected on public paths and user tokens on internal paths; internal paths MUST
  require the caller's specific scope.
- **FR-020**: There MUST be no test-only authentication bypass; tests authenticate with tokens
  issued by the mock identity provider.

**Configuration and secrets (PLT-6)**

- **FR-021**: Configuration MUST come from committed per-environment files (`local`, `test`,
  `production`), overridable at runtime by remote configuration in production; secrets MUST come
  only from environment variables or the secret store and MUST NOT be committed.
- **FR-022**: All configuration MUST be validated at startup; the service MUST refuse to start if
  a value is missing or invalid, if an address points at `localhost` outside `local` and `test`
  (unless that setting is declared in committed config as a sidecar endpoint, with a reason), or if the
  configured trusted token issuer is not a Cognito issuer outside `local` and `test`. This is a
  check of configuration at startup, not of incoming tokens.

**Observability (PLT-7)**

- **FR-023**: Every line written by the application's logging MUST be structured JSON on standard
  output only. Output from the JVM itself (e.g. runtime warnings) may be plain text and MUST NOT be
  suppressed just to keep the logs uniform.
  Every line written while handling a request MUST carry that request's ID and trace ID (and the
  internal user ID once users exist), propagated across asynchronous boundaries. Lines written
  outside a request (startup, shutdown, scheduled work) may carry neither. Every request MUST
  produce one INFO completion line with the HTTP method, the route template, the status and the
  duration, and never the actual URL values, query string, headers or bodies.
- **FR-024**: An automated test MUST verify that amounts, balances, payees, memos, account names or
  numbers, tokens, session IDs and emails never appear in log output, and that request and
  response bodies are never logged.
- **FR-025**: The service MUST emit request metrics and distributed traces, with no financial
  values in metric tags or span attributes.
- **FR-026**: Only liveness, readiness and info operational endpoints MUST be exposed, and only on
  a separate management port that is not publicly reachable.

**Packaging and local stack (PLT-8)**

- **FR-027**: The build MUST produce a container image of the service.
- **FR-028**: A single Compose stack MUST start PostgreSQL (same major version as production), the
  API, the latest released UI, and a mock OIDC provider that supports both the user sign-in flow
  and the client-credentials flow; it MUST be used for local development and by both
  repositories' end-to-end CI. The API runs in the `local` environment for local development and
  in the `test` environment in CI.
- **FR-029**: A dev-only seed MUST be available for the local stack, MUST be runnable only in the
  `local` environment, and MUST NOT be shipped as part of the production image.

**CI (PLT-9)**

- **FR-030**: GitHub Actions MUST run the single verify command on every pull request, enforcing
  every gate in the constitution's Development Workflow section, and the result MUST be required
  for merging to `main`.
- **FR-031**: Dependency and container-image vulnerability scans MUST run on every pull request and
  on a schedule against `main`, failing on high or critical findings unless suppressed with a
  comment and a re-check date; automated dependency-update pull requests MUST be enabled.

**Environments (PLT-10)**

- **FR-032**: The service MUST support exactly the `local`, `test` and `production` environments,
  each with its own committed configuration file.
- **FR-033**: The container image and `production` configuration MUST be ready to run on an AWS
  container service against a managed PostgreSQL database: no hard-coded environment details,
  secrets read from the environment or the secret store, runtime config overrides supported, and
  the management port separate so it can be kept non-public. Setting up the AWS infrastructure and
  deploying to it are deferred to a later spec.

### Key Entities

- **Public contract**: the versioned OpenAPI document the UI consumes; at this stage contains only
  shared components (error schema, security scheme, API version).
- **Internal contract**: the OpenAPI document for service-caller endpoints; never published to the
  UI.
- **Published artifact**: a uniquely versioned bundle of the public contract plus the shared
  fixtures, produced on each merge to `main`.
- **Problem Details error**: type, title, status, stable `code`, request ID, optional per-field
  errors (field name and code only).
- **Request context**: request ID, trace ID, and (later) internal user ID, carried through each
  request.
- **Environment configuration**: per-environment settings (`local`, `test`, `production`), runtime
  overrides and secrets references, validated at startup.
- **Test-only owned record**: a record with owner ID and version, used only in tests to prove the
  ownership and version mechanisms; never present in production schema.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From a fresh clone, a developer with only the documented prerequisites reaches a
  healthy local stack in under 15 minutes, using no more than two commands (build, start).
- **SC-002**: 100% of responses from every path, including errors and unknown paths, carry
  `Cache-Control: no-store` and a request ID, verified by automated tests.
- **SC-003**: 100% of error responses produced by the platform are valid Problem Details with a
  stable code, verified against the contract by automated tests.
- **SC-004**: For each gate in the constitution's Development Workflow section, a deliberately
  violating change is blocked from merging (one verified violation per gate).
- **SC-005**: A new contract version is downloadable by the UI repository within 15 minutes of
  each merge to `main`.
- **SC-006**: The log privacy test finds zero occurrences of the sensitive sample values across all
  captured output.
- **SC-007**: Stale-version updates and cross-owner access are rejected in 100% of the integration
  test cases that exercise them (409 `STALE_VERSION` and 404 respectively).
- **SC-008**: The service refuses to start in 100% of the tested unsafe-configuration cases
  (missing value, `localhost` outside `local` and `test`, non-Cognito trusted issuer outside `local` and `test`).
- **SC-009**: The full pull-request verification completes in under 15 minutes, so gates do not
  slow down feature work.
- **SC-010**: The service reports ready within 60 seconds of starting against a reachable,
  migrated database.

## Assumptions

- The technology choices in the constitution (Stack, Build, Deployment; API §II, §IV, §V, §VII,
  §IX, §XI) are fixed inputs for this feature and are not restated as requirements here; the plan
  will name them.
- No business endpoints, user records or budgets are created in this feature. User provisioning,
  user resolution and `USER_NOT_PROVISIONED` belong to the next spec (Users and auth). This spec
  provides token validation only.
- The ownership (PLT-5) and version (PLT-4) mechanisms are proven with test-only records and
  migrations that exist only in the test scope, since no production tables exist yet.
- The empty public contract is versioned starting at `0.1.0` with API path prefix `/api/v1`; it
  moves to `1.0.0` when the MVP is released, which turns on the blocking breaking-change check.
  Each published artifact gets a unique version (contract version plus a build identifier).
- The published artifact is a GitHub release asset in this repository, as suggested by API §IV.
- The domain coverage threshold (85%) applies even though the `domain` package is nearly empty;
  the gate is configured now and becomes meaningful with the first domain logic.
- If no UI release exists yet, the Compose stack and end-to-end job run without the UI service;
  the UI is added once `budget-app` publishes a release.
- The mock OIDC provider stands in for Cognito only in `local` and `test`, the only environments
  permitted to configure a non-Cognito trusted issuer (API §VI as amended in constitution v1.0.1).
- Both CI jobs run the API in the `test` environment: the integration tests in the verify command
  (database from Testcontainers, reached on `localhost`) and the end-to-end job on the Compose
  stack. `local` is only for a developer's machine. `localhost` addresses are therefore allowed in
  `local` and `test` only (API §VI as amended in constitution v1.0.1).
- Log retention, encryption and access (API §XI) are properties of the production platform's log
  collection, configured with the production environment.
- Rate limiting, `Idempotency-Key`, read replicas, cached totals, RLS policies themselves, load
  tests and a staging environment are out of scope (MVP §3).
- PLT-10 is only partly delivered here (decision 2026-10-12). This spec delivers the three
  environments' config, the production-ready image and startup validation. Setting up AWS
  production (RDS, the container service, Cognito, log collection, IaC, the deploy workflow), the
  ECS vs EKS choice, and the first production deploy belong to a later spec that must land before
  MVP launch.
- Dependencies: a GitHub repository with Actions enabled and branch protection on `main`. No AWS
  account or Cognito user pool is needed for this spec.
