# Quickstart & Validation: Platform Skeleton

**Feature**: 001-platform-skeleton | **Plan**: [plan.md](plan.md)

This guide runs the skeleton locally and walks through the checks that prove the spec's user
stories. The behavior expected at each step is defined in
[contracts/problem-codes.md](contracts/problem-codes.md) and [data-model.md](data-model.md).

## Prerequisites

- JDK 25 (e.g. Temurin or Homebrew `openjdk@25`) on `PATH`.
- A Docker-compatible runtime with Compose v2 (Docker Desktop, OrbStack or Colima). Testcontainers
  and the Compose stack both need it.
- One-time `/etc/hosts` entry so the browser, containers and API agree on the token issuer
  (research R12):

  ```text
  127.0.0.1 mock-oidc
  ```

- One-time Git hook setup, so formatting runs on commit:

  ```bash
  git config core.hooksPath .githooks
  ```

- Optional: set `NVD_API_KEY` to make the dependency scan faster. Locally you can skip the scan
  with `-Ddependency-check.skip=true`. CI never skips it.

## 1. Build and verify (Story 1 #1, Story 4)

```bash
./mvnw verify
```

**Expected**: `BUILD SUCCESS`. The log shows each gate running:

- unit and integration tests (Testcontainers Postgres and mock OIDC);
- JaCoCo check (`domain`);
- ArchUnit;
- Spotless check;
- Error Prone/NullAway;
- Redocly lint;
- oasdiff (skipped with a message if `origin/main` has no baseline);
- dependency-check.

## 2. Start the local stack (Story 1 #2)

```bash
docker compose up --build --wait
```

**Expected**:

- `postgres`, `mock-oidc` and `api` are healthy (`ui` is absent until `budget-app` publishes a
  release; enable it later with `--profile ui`).
- The API log is JSON lines and shows Flyway applying `V1__baseline`.

## 3. Health and info only on the management port (Story 1 #3–4, FR-026)

```bash
curl -s localhost:8081/actuator/health/readiness   # {"status":"UP"}
curl -s localhost:8081/actuator/health/liveness    # {"status":"UP"}
curl -s localhost:8081/actuator/info               # build + git info
curl -s -o /dev/null -w '%{http_code}\n' localhost:8081/actuator/env     # 404
curl -s -o /dev/null -w '%{http_code}\n' localhost:8080/actuator/health  # 401 (app port: auth first, then 404)
```

## 4. Platform contract on every response (Story 2)

```bash
# No token → 401 Problem Details, no-store, generated request ID
curl -si localhost:8080/api/v1/anything

# Caller-supplied request ID is echoed
curl -si -H 'X-Request-Id: demo-123' localhost:8080/api/v1/anything | grep -i x-request-id

# Malformed request ID is replaced
curl -si -H 'X-Request-Id: bad id with spaces' localhost:8080/api/v1/anything | grep -i x-request-id
```

**Expected**:

- Each response has `Cache-Control: no-store`.
- Each body is `application/problem+json` with `code: UNAUTHENTICATED` and the same `requestId`
  as the header.

Get a user token from the mock provider and call an unknown path:

```bash
TOKEN=$(curl -s -X POST http://mock-oidc:8090/budgit/token \
  -d grant_type=client_credentials -d client_id=budgit-web -d client_secret=local \
  -d scope=openid | jq -r .access_token)
curl -si -H "Authorization: Bearer $TOKEN" localhost:8080/api/v1/does-not-exist
```

**Expected**: 404, `code: NOT_FOUND`. The client IDs, and the claims that make a token
"user-facing" for local use, are in `docker/mock-oidc/config.json`.

A service token (client `budgit-provisioner`) on `/api/v1/...` must return 401 with the identical
`UNAUTHENTICATED` body.

## 5. Ownership and versions (Story 5)

These are proven by integration tests against test-only records (research R14):

```bash
./mvnw verify -Dit.test='OwnershipAndVersionIT' -Dtest=NONE -Dsurefire.failIfNoSpecifiedTests=false
```

**Expected**: the following cases pass:

- stale version → 409 `STALE_VERSION`, row unchanged;
- current version → `version + 1`;
- another user's record → 404, with a body identical to a missing record's apart from `requestId`;
- repeated create with the same ID → existing record;
- foreign ID on create → 409 `ID_CONFLICT`;
- the app role cannot run DDL.

## 6. Observability and privacy (Story 6)

```bash
./mvnw verify -Dit.test='LogPrivacyIT' -Dtest=NONE -Dsurefire.failIfNoSpecifiedTests=false
docker compose logs api | head -5   # every line is JSON with requestId and trace.id
```

## 7. Configuration safety (Story 7)

```bash
# Production profile with a localhost database → refuses to start, names the property
docker compose run --rm -e SPRING_PROFILES_ACTIVE=production -e BUDGIT_ENVIRONMENT=PRODUCTION \
  -e BUDGIT_DB_HOST=localhost api
# Production profile with the mock issuer → refuses to start (non-Cognito issuer)
```

**Expected**: the process exits non-zero, and the failure analysis names `budgit.*` properties
without printing secret values. The full production-shaped success case runs in the CI `image`
job (research R21).

## 8. Dev seed (Story 1 #5)

```bash
docker compose --profile seed run --rm seed                       # local: succeeds (no data yet)
docker compose -f compose.yaml -f compose.ci.yaml --profile seed run --rm seed   # test: guard aborts
```

## 9. CI and publishing (Stories 3–4)

One-time repository settings on GitHub:

- **Branch protection on `main`**: require a pull request, and require the status checks
  `verify`, `image` and `e2e`.
- **Secrets**: `NVD_API_KEY`.
- **Workflow permissions**: `contents: write` for `publish-contract.yml` only.

After the first merge to `main`, the releases page shows `contract-v1.0.<run>` with
`budgit-api-1.0.<run>.yaml` and `shared-fixtures-1.0.<run>.tar.gz`. The YAML has no `/internal`
paths.

To check that a breaking change is blocked, open a PR that removes a component schema still
referenced elsewhere (or, once endpoints exist, a field). Expected result: `verify` fails at the
oasdiff step unless `info.version`'s major number is bumped.
