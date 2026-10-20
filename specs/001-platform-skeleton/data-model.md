# Data Model: Platform Skeleton

**Feature**: 001-platform-skeleton | **Date**: 2026-10-12 | **Plan**: [plan.md](plan.md)

The skeleton creates **no business tables**. This document defines the database roles, the schema
conventions every later table follows, the test-only record that proves those conventions, and
the non-database models (configuration, request context, published artifact).

## 1. Database roles and schema

| Object | Kind | Properties | Used by |
| --- | --- | --- | --- |
| `budgit` | database | Owned by `budgit_owner` | n/a |
| `budgit_owner` | role | LOGIN, NOSUPERUSER, owns database and schema `budgit` | Flyway (JDBC) only |
| `budgit_app` | role | LOGIN, NOSUPERUSER, NOCREATEDB, NOCREATEROLE; no CREATE on schema | R2DBC (request path) |
| `budgit` | schema | Owner `budgit_owner`; `USAGE` granted to `budgit_app` | All application tables |

**Default privileges** (set in migration `V1__baseline.sql`):
`SELECT, INSERT, UPDATE, DELETE` on tables, and `USAGE, SELECT` on sequences, in schema `budgit`,
for every object created by `budgit_owner`, granted to `${appRole}`.

**Database setting**: `budgit.environment` (`local` | `test`), set with `ALTER DATABASE`, so
that SQL scripts such as the dev seed guard can check where they run. It is not set in
production.

Roles are created outside Flyway (cluster-level), by `docker/postgres/init/01-roles.sql` locally
and in tests. In AWS they are created by the deferred infrastructure spec.

### Migration rules (for every later spec)

- Files `src/main/resources/db/migration/V{n}__{description}.sql`, forward-only, run by
  `budgit_owner`.
- Each migration is compatible with the previous app version (expand now, contract later).
- A migration never creates roles, and never grants to anyone except `${appRole}`.

## 2. Table conventions (every user-owned, mutable table)

| Column | Type | Rule |
| --- | --- | --- |
| `id` | `uuid` PK | Client may supply on create; server generates otherwise. Exposed as string. |
| `user_id` | `uuid NOT NULL` | Owning user's internal ID, on **every** user-owned table (RLS-ready), even when also reachable via a parent. FK to `users(id)` once that table exists (next spec). |
| `budget_id` | `uuid NOT NULL` | On budget-scoped tables (later specs). |
| `version` | `integer NOT NULL DEFAULT 1` | Incremented by every update via `WHERE version = :expected`. |
| `created_at` | `timestamptz NOT NULL` | From the injected `Clock`, UTC. |
| `updated_at` | `timestamptz NOT NULL` | From the injected `Clock`, UTC. Updated with `version`. |

Indexes: a composite index leading with the owner column for every owner-scoped lookup, e.g.
`(user_id, id)`, checked with `EXPLAIN` in review (API §X).

## 3. Test-only record: `owned_widget`

This table exists only in integration tests (`src/test/resources/db/test-schema.sql`, applied by
the owner role after Flyway). It is never a migration and never present in production.

| Column | Type | Validation |
| --- | --- | --- |
| `id` | `uuid` PK | n/a |
| `user_id` | `uuid NOT NULL` | n/a |
| `name` | `text NOT NULL` | 1–100 characters |
| `version` | `integer NOT NULL DEFAULT 1` | n/a |
| `created_at`, `updated_at` | `timestamptz NOT NULL` | n/a |

**Operations** (test-only endpoint `/api/v1/test-widgets`, test-only contract):

| Operation | Behavior proven | Spec |
| --- | --- | --- |
| `POST /api/v1/test-widgets` (create, optional `id` in body) | No `id` → server generates a UUID → 201; new `id` → 201; `id` exists for the same owner → existing record, unchanged (200); `id` belongs to another owner → 409 `ID_CONFLICT` | FR-016, Story 5 #5 |
| `GET /api/v1/test-widgets/{id}` | Own → 200; other user's or missing → 404 (identical body) | FR-017, Story 5 #3 |
| `PATCH /api/v1/test-widgets/{id}` with `version` | Current version → 200 with `version + 1`; stale → 409 `STALE_VERSION`, row unchanged; other user's → 404 | FR-015, Story 5 #1–3 |
| `DELETE /api/v1/test-widgets/{id}?version=` | Same version and ownership rules | FR-015, FR-017 |

**State transitions**: `version` 1 → 2 → … (strictly +1 per successful update); a delete
removes the row (hard delete).

## 4. Shared typed IDs (`common`)

| Type | Wraps | Purpose |
| --- | --- | --- |
| `UserId` | `UUID` | Internal user ID; required parameter of owner-scoped persistence methods |
| `BudgetId` | `UUID` | Budget ID; alternative owner parameter for budget-scoped tables |
| `RequestId` | `String` | Validated request ID (pattern `^[A-Za-z0-9._-]{1,64}$`) |

The ArchUnit ownership rule keys off `UserId`/`BudgetId` parameters (research R13).

## 5. Platform exceptions → Problem codes

| Exception (`common`) | Status / code |
| --- | --- |
| `NotFoundException` | 404 `NOT_FOUND` |
| `StaleVersionException` | 409 `STALE_VERSION` |
| `IdConflictException` | 409 `ID_CONFLICT` |
| `BusinessRuleException` | 422 `BUSINESS_RULE_VIOLATION` (or a specific code) |
| `UpstreamFailedException` / `UpstreamTimeoutException` | 502 `UPSTREAM_FAILED` / 504 `UPSTREAM_TIMEOUT` |

The full mapping and headers are in [contracts/problem-codes.md](contracts/problem-codes.md).

## 6. Configuration model (`config`)

`BudgitProperties` (`@Validated @ConfigurationProperties("budgit")`, record), with
`@SafeForEnvironment` at class level:

| Property | Type | Constraint |
| --- | --- | --- |
| `budgit.environment` | enum `LOCAL`/`TEST`/`PRODUCTION` | required; must match the active profile |
| `budgit.security.issuer-uri` | URI | required; must match the Cognito issuer pattern outside `LOCAL`/`TEST` |
| `budgit.security.jwk-set-uri` | URI | optional (derived from issuer when absent) |
| `budgit.security.user-client-ids` | set of strings | required, non-empty |
| `budgit.security.service-client-ids` | set of strings | required (may be empty until the first internal endpoint) |
| `budgit.security.clock-skew` | duration | default 60 s, max 5 min |
| `budgit.security.jwks-timeout` | duration | default 3 s |
| `budgit.http.max-request-size` | data size | default 256 KB |
| `budgit.sidecar-endpoints` | map: property name → reason | optional; each key names a setting allowed to point at `localhost` because a sidecar serves it; reason required (non-blank) |

Spring-owned properties validated by the same rule set: R2DBC URL, Flyway URL, and OTLP
endpoints. No host may be `localhost` or a loopback address outside `LOCAL`/`TEST`, except a
setting listed in `budgit.sidecar-endpoints` (API §VI, constitution v1.1.0). Example, once the
infrastructure spec adds a telemetry collector sidecar:

```yaml
budgit:
  sidecar-endpoints:
    management.opentelemetry.tracing.export.otlp.endpoint: "ADOT collector sidecar in the ECS task"
```

**Secrets** (environment variables or Secrets Manager, never committed): app-role DB password,
owner-role DB password.

## 7. Request context

| Field | Source | Carried in | Appears in |
| --- | --- | --- | --- |
| `requestId` | `X-Request-Id` or generated | Reactor `Context` and tracing baggage | Response header, Problem body, every log line |
| `traceId` / `spanId` | Micrometer Tracing (OTel) | Tracing context | Log lines, traces |
| `userId` | Added by the next spec (internal UUID, never the Cognito `sub`) | Tracing baggage | Log lines, span attribute; never a metric tag |

## 8. Published contract artifact

| Asset | Content |
| --- | --- |
| `budgit-api-{major}.{minor}.{run}.yaml` | Bundled public contract, `info.version` rewritten to the release version |
| `shared-fixtures-{major}.{minor}.{run}.tar.gz` | `src/test/resources/shared-fixtures/` (contains only `README.md` for now) |

GitHub release tag: `contract-v{major}.{minor}.{run}`. Never contains `budgit-internal-api.yaml`.
