## Summary

This PR implements two new features on the Task Management API: a `search` query parameter for `GET /tasks` that performs case-insensitive substring filtering on `title` and `description`, and a new `GET /tasks/summary` endpoint that returns aggregate task counts broken down by status and priority. Both features are scoped to the authenticated user's tasks, conform to existing API envelope and error-handling conventions, and ship with full unit test coverage as specified in FR-10.

---

## Changes Made

- **`router/routes.js`** — Registered `GET /tasks/summary` before `GET /tasks/:id` to prevent the literal string `"summary"` from being matched as a task ID parameter (DD-09 risk mitigation; T-2)
- **`db/query-builder.js`** — Added `withSearch(term)` predicate method generating a parameterised `LOWER(title) LIKE ? OR LOWER(description) LIKE ?` clause (PostgreSQL: `ILIKE`); `%` wildcards applied inside the bound value, never via string interpolation, to prevent SQL injection (T-3)
- **`repositories/task.repository.js`** — Extended `list()` to accept an optional `search` parameter, passing it to the query builder; `user_id` scope and pagination are always applied regardless of whether a search term is present (T-4)
- **`services/task.service.js`** — Updated `list()` signature to accept and forward `search` alongside `status`, `priority`, and pagination parameters to the repository (T-5)
- **`controllers/task.controller.js`** — Extracts and validates the `search` query parameter (trims whitespace, rejects empty-after-trim and >200-character values with a `400` and standard error payload), then forwards to the service; validation errors are logged via the existing logger (T-6)
- **`repositories/summary.repository.js`** *(new)* — Implements `getSummary(userId)` using three parameterised aggregate queries (`GROUP BY status`, `GROUP BY priority`, total count) scoped to `user_id`; returns raw bucket maps (T-7)
- **`services/summary.service.js`** *(new)* — Calls the summary repository and zero-fills any missing status (`pending`, `in_progress`, `complete`) or priority (`low`, `medium`, `high`) buckets to guarantee a consistent DTO shape regardless of data sparsity (T-8)
- **`controllers/summary.controller.js`** *(new)* — Handles `GET /tasks/summary`; reads `user_id` from auth context injected by existing middleware, delegates to summary service, wraps result in the standard response envelope, returns `200 OK`; logs and returns `500` on unexpected errors (T-9)
- **`migrations/add_search_indexes.sql`** *(new)* — Non-blocking migration script adding indexes on `tasks.title` and `tasks.description`; includes inline comment documenting that `LIKE '%term%'` does not benefit from B-tree indexes and recommending GIN/full-text index evaluation at scale (T-10)
- **`tests/unit/search.test.js`** *(new)* — Unit tests covering FR-10 cases a–g: title match, description match, case-insensitive match, no-match empty result, search + status, search + priority, search + pagination; repository calls mocked; all tests assert `user_id` scope predicate presence (T-11)
- **`tests/unit/summary.test.js`** *(new)* — Unit tests covering FR-10 cases h–j: mixed dataset counts, empty task set all-zero response, zero-valued bucket completeness; additional controller-layer tests for `401` enforcement and `500` on repository failure (T-12)
- **`docs/openapi.yaml`** — Added `search` parameter definition on `GET /tasks` and full `GET /tasks/summary` endpoint schema including all bucket fields, `400`/`401` error schemas, and a note on the full-table-scan caveat for large datasets (T-14/T-15)

---

## Test Evidence

All unit tests in `tests/unit/search.test.js` and `tests/unit/summary.test.js` pass (FR-10 cases a–j fully covered). The full existing test suite remains green. End-to-end validation was performed against a seeded test database confirming:

- `GET /tasks?search=<term>` returns correctly filtered, user-scoped, paginated results
- Combined filters (`search + status`, `search + priority`) behave as logical AND
- `GET /tasks/summary` returns correct counts for a known mixed dataset and all-zero counts for an empty dataset
- Both endpoints return `401` for unauthenticated requests and `400` for invalid parameter input
- Response envelopes match existing API conventions
- Errors appear in log output in the same format as existing endpoints

---

## Known Limitations

- **Full-table-scan risk:** `LIKE '%term%'` substring search cannot use standard B-tree indexes. The migration in `add_search_indexes.sql` is provided but marked non-blocking. For datasets expected to grow significantly, a GIN or full-text index (PostgreSQL) should be evaluated before enabling this feature at scale. This is documented in both the migration script and the OpenAPI docs per NFR-05.
- **`GET /tasks/summary` is unfiltered:** The summary always reflects all tasks accessible to the authenticated user. Filtering the summary by status, priority, or any other parameter is explicitly out of scope for this iteration.
- **Search field scope:** Only `title` and `description` are searched. Tags, comments, assignee names, and other fields are out of scope.
- **No fuzzy/phonetic matching:** Search is strict substring only; typos and stemmed variants will not match.
- **No caching contract:** No cache-control headers are set on `/tasks/summary`; caching strategy is deferred per the clarifying Q&A.

---

## Reviewer Checklist

- [ ] `GET /tasks/summary` route is registered **before** `GET /tasks/:id` in the router — confirm `summary` cannot be matched as a task ID
- [ ] `withSearch()` in `query-builder.js` uses parameterised queries exclusively; `%` wildcards are inside the bound value, not interpolated into the SQL string
- [ ] `user_id` scope predicate is applied in **both** `task.repository.js` and `summary.repository.js` on every query path, including when `search` is absent
- [ ] Zero-fill logic for missing status/priority buckets lives in `summary.service.js`, not in the repository or controller
- [ ] Existing auth middleware is wired to `GET /tasks/summary` in the router; no unauthenticated access is possible
- [ ] `search` parameter validation correctly rejects empty-after-trim values and values exceeding 200 characters with a `400` and standard error payload
- [ ] All FR-10 test cases (a–j) are present and assertions include `user_id` scope verification
- [ ] `401` enforcement and `500` error path are tested at the summary controller layer
- [ ] Error logging on both new endpoints uses the existing logger in the same format as current endpoints (NFR-06)
- [ ] New code style, naming conventions, and envelope structure match the baseline documented in T-1 (NFR-04)
- [ ] OpenAPI spec updated under the existing version namespace with all new parameters, response schemas, and the full-table-scan caveat note
- [ ] `add_search_indexes.sql` migration is clearly marked non-blocking and includes the GIN/full-text index recommendation comment