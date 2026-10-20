# Problem Codes and Platform HTTP Behavior

Registry of stable `code` values in Problem Details bodies ([budgit-api.yaml](budgit-api.yaml),
`ProblemCode`). Codes are never renamed or reused. A new code is an additive contract change.

| Code | Status | Raised when | Body notes |
| --- | --- | --- | --- |
| `VALIDATION_FAILED` | 400 | Bean Validation failure, malformed JSON, unknown JSON field, bad path/query format (`YYYY-MM`, UUID) | `errors[]` with `field` + constraint `code`; never the value |
| `UNAUTHENTICATED` | 401 | Missing token, bad signature, expired, wrong issuer, not an access token, client not allowed on this path kind, missing scope (internal) | Identical body for every cause |
| `NOT_FOUND` | 404 | Unknown path; record missing; record owned by another user | Identical body for every cause |
| `METHOD_NOT_ALLOWED` | 405 | Known path, unsupported method | Framework-produced; kept as Problem Details |
| `NOT_ACCEPTABLE` | 406 | `Accept` excludes JSON / problem+json | Framework-produced |
| `STALE_VERSION` | 409 | Update/delete with a `version` that is not current | No current values in the body |
| `ID_CONFLICT` | 409 | Create with a client ID that belongs to a different record | |
| `PAYLOAD_TOO_LARGE` | 413 | Body over the configured limit | |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | Body content type not supported | |
| `BUSINESS_RULE_VIOLATION` | 422 | Domain rule rejected the request (used by later specs; may be refined into specific codes) | |
| `RATE_LIMITED` | 429 | Reserved; rate limiting is deferred (MVP §3) | |
| `INTERNAL_ERROR` | 500 | Any unhandled exception | Generic title, no detail |
| `UPSTREAM_FAILED` | 502 | External call returned an error | |
| `SERVICE_UNAVAILABLE` | 503 | Database unavailable; pool exhausted; shutting down; JWKS unreachable during authentication | `Retry-After` when known |
| `UPSTREAM_TIMEOUT` | 504 | External call timed out | |

Added by the next spec (Users and auth): `USER_NOT_PROVISIONED` (401). It's returned only after a
token has fully validated, and the UI treats it differently from `UNAUTHENTICATED`.

## Create with a client-supplied ID

Every resource has one create operation: `POST` to the collection (e.g. `POST /api/v1/accounts`),
with an optional `id` (UUID) in the body. Without an `id`, the server generates one.

`PUT` is not used for creates. Updates use `PATCH` to change some fields of a record, or `PUT` to
replace a resource whose request body is the whole resource (e.g. a single value). Both carry the
record's `version` and return `409 STALE_VERSION` when it is stale (API §V).

Applies to every create endpoint (API §V, spec FR-016):

| Situation | Response |
| --- | --- |
| No record with this ID exists | `201 Created`, body = the new record |
| A record with this ID already exists for the same owner (e.g. a retry) | `200 OK`, body = the existing record, unchanged |
| The ID belongs to a different record (e.g. another user's) | `409` Problem Details, code `ID_CONFLICT` |

Clients can treat both `201` and `200` as success and use the returned body; the status tells them
whether this request created the record.

## Headers on every response (application port)

| Header | Value |
| --- | --- |
| `X-Request-Id` | Incoming value if it matches `^[A-Za-z0-9._-]{1,64}$`, else a generated UUID |
| `Cache-Control` | `no-store` (exactly) |
| `Content-Type` (errors) | `application/problem+json` |

## Path kinds

| Prefix | Caller | Token requirement | Contract |
| --- | --- | --- | --- |
| `/api/v1/**` | User-facing (Next.js server) | Cognito access token, `client_id` in user-facing clients | `budgit-api.yaml` (published) |
| `/internal/v1/**` | Service callers | Cognito client-credentials token, `client_id` in service clients, required scope | `budgit-internal-api.yaml` (not published) |
| anything else | n/a | Token checked like `/api/**` first, then 404 | n/a |

Management endpoints (`/actuator/health/liveness`, `/actuator/health/readiness`, `/actuator/info`)
are served **only** on the management port (8081). They are unauthenticated there and absent from
the application port.
