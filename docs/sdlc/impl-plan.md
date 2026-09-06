# Implementation Plan — Task Management API: Search & Summary

## Tasks

1. **T-1: Audit & Document Existing Codebase Conventions** — Review the existing Task controller, service, repository, query builder, response envelope serializer, error handler, and logger implementations. Document coding patterns, naming conventions, envelope structure, error payload format, auth middleware contract (how `user_id` is injected into request context), and the test framework setup. This output becomes the reference standard for all subsequent tasks.
   Depends on: None
   Est: 3h

2. **T-2: Verify & Register Route Order for `GET /tasks/summary`** — Inspect the existing API router configuration and confirm the order in which `GET /tasks/:id` (or equivalent parameterised route) is registered. Register the new `GET /tasks/summary` route explicitly before any parameterised task ID route to prevent the literal string `summary` from being matched as an ID (DD-09). Document the change in a comment adjacent to the route declarations.
   Depends on: T-1
   Est: 1h

3. **T-3: Extend Query Builder with Case-Insensitive Substring Predicate** — Add a new predicate method to the existing query builder that accepts a `search` term and generates a parameterised SQL clause of the form `(LOWER(title) LIKE LOWER(:term) OR LOWER(description) LIKE LOWER(:term))` (or dialect-native `ILIKE` for PostgreSQL). The term must be wrapped with `%` wildcards inside the parameterised value, not interpolated into the SQL string, to prevent SQL injection. The method must compose cleanly with existing predicates (status, priority, user scope) as a logical AND.
   Depends on: T-1
   Est: 3h

4. **T-4: Extend Task Repository with Search Filter Support** — Update the existing task repository's list query method to accept an optional `search` parameter and pass it through to the query builder predicate added in T-3. Ensure the `user_id` scope predicate is always applied regardless of whether a search term is present. Verify that pagination (LIMIT/OFFSET) is applied after all filter predicates.
   Depends on: T-3
   Est: 2h

5. **T-5: Extend Task Service with Search Orchestration** — Update the existing task service's list method signature to accept an optional `search` string alongside existing `status`, `priority`, and pagination parameters. Pass all parameters through to the repository call added in T-4. No business logic transformation is required; the service layer is responsible for composing the full filter criteria object and delegating to the repository.
   Depends on: T-4
   Est: 2h

6. **T-6: Extend Task Controller with `search` Parameter Extraction and Validation** — Update the existing task controller's handler for `GET /tasks` to extract the `search` query parameter. Apply validation: reject with a `400` response using the standard error payload if the value is present but empty after trimming, or exceeds a defined maximum length (recommended: 200 characters, consistent with field length conventions identified in T-1). Pass validated parameter to the task service (T-5). Log validation errors via the existing logger.
   Depends on: T-5
   Est: 2h

7. **T-7: Implement Summary Repository** — Create a new Summary Repository following the patterns identified in T-1. Implement a single method that accepts `user_id` and executes two `GROUP BY` aggregate SQL queries scoped to that user: (a) `SELECT status, COUNT(*) FROM tasks WHERE user_id = :uid GROUP BY status` and (b) `SELECT priority, COUNT(*) FROM tasks WHERE user_id = :uid GROUP BY priority`, plus a total count query. Return a raw bucket map (e.g., `{ status: { pending: N, ... }, priority: { low: N, ... }, total: N }`) to the caller. Use parameterised queries throughout.
   Depends on: T-1
   Est: 3h

8. **T-8: Implement Summary Service** — Create a new Summary Service following existing service patterns. Implement a method that accepts `user_id`, calls the Summary Repository (T-7), and merges the returned raw bucket map into a fixed-shape DTO that guarantees all defined status values (`pending`, `in_progress`, `complete`) and priority values (`low`, `medium`, `high`) are present with a default of `0` for any bucket not returned by the database. This zero-fill logic must live in the service, not the repository or controller.
   Depends on: T-7
   Est: 2h

9. **T-9: Implement Summary Controller** — Create a new Summary Controller following existing controller patterns. Implement a handler for `GET /tasks/summary` that reads `user_id` from the auth context (as injected by the existing auth middleware), delegates to the Summary Service (T-8), wraps the resulting DTO in the standard response envelope, and returns `200 OK`. Handle and log any service or repository errors using the existing error handler and logger, returning a `500` response with the standard error payload on failure. Return `401` for unauthenticated requests (enforced by the existing auth middleware — verify this is applied to the new route).
   Depends on: T-8, T-2
   Est: 2h

10. **T-10: Recommend and Script Database Index Additions** — Produce a migration script (or index recommendation document, consistent with the project's schema migration tooling) that adds indexes on the `title` and `description` columns of the `tasks` table to mitigate full-table-scan risk for substring search (NFR-05). Mark the migration as recommended but non-blocking. Include a comment in the script and a note in the repository's documentation that `LIKE '%term%'` queries do not benefit from standard B-tree indexes and that full-text index types (e.g., GIN for PostgreSQL) should be evaluated if dataset size grows.
    Depends on: T-1
    Est: 2h

11. **T-11: Write Unit Tests — Search Functionality** — Using the existing test framework and conventions (T-1), implement the following unit test cases as specified in FR-10: (a) search match on `title`, (b) search match on `description`, (c) case-insensitive matching (uppercase term matches lowercase stored value), (d) no-match returning an empty result set, (e) search combined with `status` filter, (f) search combined with `priority` filter, (g) search combined with pagination parameters. Tests should cover the service and query builder layers with repository calls mocked. Verify that `user_id` scoping is present in all generated queries.
    Depends on: T-6
    Est: 4h

12. **T-12: Write Unit Tests — Summary Functionality** — Using the existing test framework and conventions (T-1), implement the following unit test cases as specified in FR-10: (h) summary with a mixed dataset returning correct non-zero counts per bucket, (i) summary with an empty task set returning all-zero counts, (j) summary always returning all defined status and priority buckets even when zero. Tests should cover the service layer with the repository mocked. Additionally test the controller layer for correct `401` enforcement and correct `500` handling on repository failure.
    Depends on: T-9
    Est: 3h

13. **T-13: Integration & End-to-End Validation** — Execute the full test suite against a running instance with a seeded test database. Validate: `GET /tasks?search=<term>` returns correctly filtered, paginated, user-scoped results; combined filters (`search + status`, `search + priority`) behave as logical AND; `GET /tasks/summary` returns the correct counts for a known dataset including zero-valued buckets; both endpoints return `401` for unauthenticated requests; both endpoints return `400` for invalid parameter input; response envelopes match existing API conventions; errors appear in log output in the same format as existing endpoints.
    Depends on: T-11, T-12
    Est: 3h

14. **T-15: Update API Documentation** — Update the existing API reference documentation (OpenAPI spec or equivalent) to include: the `search` query parameter on `GET /tasks` with type, constraints, and example; the new `GET /tasks/summary` endpoint with full request/response schema including all status and priority buckets; `400` and `401` error response schemas; and a note on the full-table-scan caveat for large datasets. Ensure the documentation is published under the same API version namespace as all other endpoints (NFR-01).
    Depends on: T-13
    Est: 2h

---

## Milestones

| Milestone | Tasks | Deliverable |
|---|---|---|
| M-1: Foundation & Standards Baseline | T-1 | Documented codebase conventions, envelope structure, auth contract, and test framework reference used as the standard for all implementation work |
| M-2: Search Feature Complete | T-2, T-3, T-4, T-5, T-6 | `GET /tasks?search=<term>` fully functional with case-insensitive substring matching, composable with existing filters and pagination, `user_id`-scoped, with `400`/`401` error handling |
| M-3: Summary Feature Complete | T-7, T-8, T-9 | `GET /tasks/summary` endpoint live, returning consistently shaped DTO with zero-filled buckets, `user_id`-scoped, with `401`/`500` error handling and logging |
| M-4: Performance Risk Addressed | T-10 | Index migration script and documented full-table-scan caveat available for review and deployment decision |
| M-5: Test Coverage Complete | T-11, T-12 | All 10 FR-10 unit test cases passing plus controller-layer error path tests; full test suite green |
| M-6: Validated & Shipped | T-13, T-14 | End-to-end validation passed against seeded database; API documentation updated and published under existing version namespace; feature ready for production deployment |

---

## Risk Mitigations

| Risk | Mitigation | Owner |
|---|---|---|
| `GET /tasks/summary` route matched as a task ID by parameterised `GET /tasks/:id` route, returning 404 or incorrect DB lookup | T-2 explicitly audits and enforces route registration order, placing the summary route before any parameterised route; add a regression test asserting the correct handler is invoked | Backend Lead |
| Substring search with `LIKE '%term%'` causes full-table scans and degrades performance as dataset grows | T-10 produces an index migration script with a GIN/full-text index recommendation for PostgreSQL; caveat is documented in both code and API docs; NFR-05 classifies this as a known accepted risk for current scope | Backend Lead |
| SQL injection via the `search` query parameter | T-3 mandates parameterised queries with `%` wildcards applied inside the bound parameter value, never via string interpolation; this is enforced in code review and validated in T-11 tests | Backend Lead |
| Zero-count buckets omitted from summary response if the database returns no rows for a given status or priority value, breaking client contract | Zero-fill logic is explicitly implemented in the Summary Service (T-8) with dedicated unit tests for the empty-dataset case (FR-10i, FR-10j) and a non-nullable DTO shape | Backend Lead |
| New endpoints inadvertently expose tasks belonging to other users if `user_id` scoping is applied inconsistently across layers | `user_id` is injected by the existing auth middleware and applied at the database query layer in both the Task Repository (T-4) and Summary Repository (T-7); T-11 and T-12 include explicit assertions that user-scope predicates are present in all generated queries | Backend Lead / Security |
| New code diverges from existing conventions, increasing maintenance burden or failing code review | T-1 produces an explicit conventions reference before any implementation begins; all new components are reviewed against this reference; NFR-04 compliance is a code review gate criterion | Tech Lead |
| Auth middleware not applied to the new `GET /tasks/summary` route, allowing unauthenticated access | T-9 explicitly verifies that the existing auth middleware is wired to the new route; T-12 includes a `401` enforcement test for the summary controller | Backend Lead |
| Incomplete test coverage if FR-10 test cases are split across tasks and some are missed | FR-10 test cases are mapped explicitly to T-11 (cases a–g) and T-12 (cases h–j) with a checklist; T-13 integration validation confirms the full suite is green before the milestone is closed | Tech Lead |