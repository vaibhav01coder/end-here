# Requirements — Task Management API: Search & Summary

## User Story
As a user, I want to search and summarize tasks so I can quickly find tasks by keyword and see status/priority counts at a glance.

## Clarifying Q&A

| # | Question | Answer |
|---|----------|--------|
| 1 | Should the keyword search support partial/substring matching, exact phrase matching, or both, and should it handle special characters or wildcards? | Not answered — defaulting to case-insensitive substring/partial matching; wildcard and special character support assumed out of scope |
| 2 | Should `GET /tasks/summary` respect filters (e.g., status, priority, user/owner) so users can get a summary of a filtered subset, or should it always reflect all tasks? | Not answered — defaulting to unfiltered summary reflecting all tasks accessible to the caller |
| 3 | Should search results and the summary endpoint be scoped to the authenticated user's tasks only, or visible across all users depending on role/permissions? | Not answered — defaulting to authenticated user's own tasks only |
| 4 | Are there performance requirements or constraints to consider, such as expected dataset size, response time SLAs, or whether database indexing on title/description is already in place? | Not answered — no specific SLAs assumed; indexing recommendations noted as implementation guidance |
| 5 | Should the summary endpoint be included in the existing API versioning scheme, and are there any caching expectations for its response? | Not answered — defaulting to inclusion in current API version; no caching contract defined |

---

## Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | The system shall support a `search` query parameter on `GET /tasks` (e.g., `GET /tasks?search=<term>`) that filters returned tasks to only those whose `title` or `description` contains the search term. |
| FR-02 | Keyword matching shall be case-insensitive and substring-based (partial match); a task is returned if the term appears anywhere within the `title` or `description` fields. |
| FR-03 | The `search` parameter shall be combinable with all existing query parameters including `status`, `priority`, and pagination parameters (`page`, `limit`, or equivalent). Combined filters shall apply as a logical AND. |
| FR-04 | The system shall expose a new endpoint `GET /tasks/summary` that returns aggregate counts for the authenticated user's accessible tasks. |
| FR-05 | The `GET /tasks/summary` response shall include: (a) `total` — total task count; (b) `by_status` — counts keyed by each status value (`pending`, `in_progress`, `complete`); (c) `by_priority` — counts keyed by each priority value (`low`, `medium`, `high`). |
| FR-06 | Status or priority buckets with zero tasks shall still be present in the summary response with a value of `0`, ensuring a consistent response shape. |
| FR-07 | Both the search filter and the summary endpoint shall be scoped to tasks owned by or assigned to the currently authenticated user; tasks belonging to other users shall not be included. |
| FR-08 | Both new endpoints shall conform to the existing API JSON response envelope conventions (e.g., consistent wrapper structure, field naming, HTTP status codes). |
| FR-09 | Both new endpoints shall follow the existing API error-handling patterns, returning appropriate error responses (e.g., `400` for invalid parameter values, `401` for unauthenticated requests) with standard error payloads. |
| FR-10 | Unit tests shall be added covering: (a) search match on title, (b) search match on description, (c) case-insensitive matching, (d) no-match returning empty result set, (e) search combined with status filter, (f) search combined with priority filter, (g) search combined with pagination, (h) summary with a mixed dataset, (i) summary with an empty task set, (j) summary returning zero-valued buckets. |

---

## Non-Functional Requirements

| ID | Category | Requirement |
|----|----------|-------------|
| NFR-01 | Consistency | New endpoints must follow existing API versioning conventions and be accessible under the same version namespace currently in use. |
| NFR-02 | Security | Both endpoints must require authentication in line with the existing authentication/authorisation mechanism; unauthenticated requests must be rejected with `401`. |
| NFR-03 | Reliability | The `GET /tasks/summary` endpoint must return a complete, consistently shaped response regardless of whether any tasks exist; zero-count buckets must never be omitted. |
| NFR-04 | Maintainability | New code must match the existing codebase's coding standards, patterns, and test framework conventions so it can be maintained without specialist knowledge. |
| NFR-05 | Performance | It is recommended (not mandated) that the `title` and `description` columns be indexed to support substring search at scale; any full-table-scan risk should be documented for future review. |
| NFR-06 | Observability | Errors on the new endpoints must be logged in the same manner as existing endpoints to ensure consistent operational monitoring. |

---

## Out of Scope

- Wildcard or regex-based search syntax
- Full-text search ranking or relevance scoring
- Filtering the `GET /tasks/summary` endpoint by status, priority, or any other query parameter
- Cross-user or admin-level visibility of other users' tasks via these endpoints
- Dedicated caching layer or cache-control headers for `/tasks/summary`
- New role or permission definitions beyond what is already implemented
- Search against fields other than `title` and `description` (e.g., tags, comments, assignee name)
- Phonetic, fuzzy, or stemmed matching
- Exposing search or summary functionality via any protocol other than the existing REST/HTTP API