# Design Review — Task Management API: Search & Summary

## Risks & Gaps Identified

### R-1 · Route Registration Order Not Enforced by Architecture

**Risk:** DD-09 documents that `GET /tasks/summary` must be registered before any parameterised `GET /tasks/:id` route, but this is stated only as a prose decision with no structural enforcement, meaning a future refactor or framework upgrade could silently break the route by treating `"summary"` as a task ID.
**Decision:** Introduce a router integration test that explicitly asserts `GET /tasks/summary` returns 200 (not 404 or a DB lookup error) as a regression guard, and add a code comment co-located with the route registration marking the ordering requirement as load-bearing.

---

### R-2 · Full-Table Scan Risk Under Load Deferred Without Threshold

**Risk:** DD-07 marks `title` and `description` indexes as "recommended, non-mandatory," but provides no dataset-size threshold or acceptance criterion that would trigger mandatory indexing, meaning the decision to add indexes could be deferred indefinitely even as the table grows to a size where NFR-05 performance targets are violated.
**Decision:** Add a concrete threshold (e.g., "indexes are mandatory before production deployment if the tasks table exceeds N rows or if the P95 latency of `GET /tasks?search=` under representative load exceeds the NFR-05 target") so the decision becomes objectively actionable rather than indefinitely advisory.

---

### R-3 · Search Input Validation Scope Is Undefined

**Risk:** The architecture specifies that the Task Controller returns a 400 on invalid parameters but does not define validation rules for the `search` term itself — no maximum length, no disallowed character classes, and no minimum length — leaving the system open to excessively long strings that stress the `LOWER()/ILIKE` query path or to inputs that degrade query plan performance.
**Decision:** Define explicit validation constraints for the `search` parameter (e.g., maximum length of 200 characters, trimming of surrounding whitespace, empty-string treated as absent) in the Task Controller specification, and add a corresponding 400 response case to the data-flow diagram.

---

### R-4 · Summary Endpoint Has No Error Handling Prior to DB Execution

**Risk:** The data-flow diagram shows the `DB error?` check only *after* the database returns results, meaning errors that occur before query execution — such as a lost database connection or a context-cancellation timeout — have no documented handling path and could result in an unhandled exception propagating to the client in an inconsistent format.
**Decision:** Add an explicit error branch at the Summary Repository execution step (mirroring the existing Task Repository path) covering pre-execution failures, and confirm that the existing Error Handler middleware is wired to catch repository-layer exceptions for the summary path as it is for the task-list path.

---

### R-5 · Pagination Is Absent From Summary Endpoint But Not Explicitly Excluded

**Risk:** The summary endpoint always returns a fixed-shape DTO today (DD-03, DD-04), but the architecture does not explicitly state that pagination, filtering, or additional breakdown dimensions (e.g., by assignee or date range) are rejected with a defined error if passed as query parameters, creating ambiguity about whether unknown query parameters are silently ignored or surfaced as validation errors.
**Decision:** Explicitly document in the Summary Controller specification that any query parameters supplied to `GET /tasks/summary` are ignored without error (or alternatively rejected with 400), and confirm this behaviour is covered by a unit test case so the contract is unambiguous for future consumers and contributors.

---

### R-6 · Database Dialect Ambiguity Creates Inconsistent Search Behaviour

**Risk:** The architecture permits either PostgreSQL or MySQL as the backing database and treats `ILIKE` (PostgreSQL-native) and `LOWER() LIKE` (portable) as interchangeable alternatives, but the two approaches have subtly different collation and locale behaviours for non-ASCII characters; if the dialect choice is made at deployment time rather than build time, the same codebase could produce different search results across environments.
**Decision:** Require the Query Builder to commit to a single, dialect-detected strategy at application startup, log the chosen approach, and add an integration test covering at least one non-ASCII search term (e.g., an accented character) to verify that case-insensitive matching behaves as expected for the deployed dialect rather than assuming ASCII-only input.

---

## Agreed Design Decisions

| ID | Decision |
|---|---|
| DD-01 | Extend `GET /tasks` with a `search` query parameter; do not introduce a separate search endpoint |
| DD-02 | Implement search via `LOWER()/LIKE` or dialect-native `ILIKE`; no external search engine |
| DD-03 | `GET /tasks/summary` always returns all status and priority buckets, injecting zero for missing values |
| DD-04 | `GET /tasks/summary` is unfiltered beyond `user_id` scope; filtered summaries are out of scope |
| DD-05 | Summary aggregation is performed via SQL `GROUP BY` in the database, not in application memory |
| DD-06 | Both endpoints are strictly scoped to the authenticated user's `user_id` at the query layer |
| DD-07 | `title`/`description` indexes are recommended; mandatory if a concrete performance threshold (to be defined per R-2) is breached before production deployment |
| DD-08 | New components follow existing layered patterns; no new abstractions introduced |
| DD-09 | `GET /tasks/summary` is registered in the router before any parameterised `GET /tasks/:id` route, enforced by a router regression test per R-1 |
| DD-10 | Error logging for new endpoints reuses the existing Logger via the same call sites as existing controllers |
| DD-R3 | `search` parameter is subject to explicit validation constraints (max length, whitespace trimming, empty-string-as-absent) defined in the Task Controller specification |
| DD-R4 | Summary Repository pre-execution failures are handled explicitly and routed through the existing Error Handler |
| DD-R5 | Unknown query parameters on `GET /tasks/summary` are handled with a documented and tested behaviour (silent ignore or explicit 400) |
| DD-R6 | Query Builder commits to a single dialect strategy at startup; integration tests cover non-ASCII search input |

---

## Architecture Updates Applied

1. **Router registration guard (R-1 / DD-09):** The component table entry for the API Router has been updated to note that route ordering is a correctness requirement, and a router-layer integration test is added to the testing specification alongside the ten existing FR-10 unit test cases.

2. **Index mandate threshold (R-2 / DD-07):** DD-07 is revised from "recommended" to "recommended up to a defined threshold, mandatory beyond it." The Database row in the component table is updated to reference this threshold, and a pre-production performance gate is added to the deployment checklist.

3. **Search parameter validation (R-3 / DD-R3):** The Task Controller component description is expanded to include `search` in its validation rules. A new 400 branch for `search`-specific validation failure is added to the data-flow diagram immediately after the existing param-validation diamond, with the constraint values (max length 200, whitespace-trimmed, empty treated as absent) noted inline.

4. **Summary pre-execution error path (R-4 / DD-R4):** The summary data-flow is updated so that the `DB error?` decision node is moved to wrap the entire Summary Repository execution step, not just the result-processing step. A failure branch from this node flows to the existing 500 / log path, matching the structural pattern already present in the task-list flow.

5. **Summary query-parameter contract (R-5 / DD-R5):** The Summary Controller component description is updated to explicitly state the handling of unexpected query parameters. The chosen behaviour (silent ignore, or 400 rejection) is reflected in the data-flow diagram and a corresponding test case is added to the FR-10 unit test list.

6. **Dialect strategy commitment (R-6 / DD-R6):** The Query Builder component description is updated to state that dialect detection occurs at startup and the selected strategy is logged. The Tech Stack table row for Query Language is revised to note that `LOWER()/LIKE` and `ILIKE` are not treated as runtime-interchangeable; the integration test suite is noted as requiring at least one non-ASCII search test per supported dialect.