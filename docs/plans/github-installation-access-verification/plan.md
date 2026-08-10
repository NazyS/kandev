---
spec: docs/specs/integrations/github-authentication.md
created: 2026-08-10
status: complete
---

# Implementation Plan: GitHub Installation Access Verification

## Overview

`GitHubOAuthClient.UserCanAccessInstallation` currently calls the undocumented
`GET /user/installations/{installation_id}` route. GitHub returns 404 for that route, Kandev maps
the 404 to `accessible=false`, and the installation service retries the same invalid request before
rejecting an otherwise valid callback. Replace that lookup with GitHub's documented paginated user
installation collection, while preserving the service's bounded retry for a valid list that does
not yet contain a newly created installation.

## Backend

### OAuth installation association client

Update `apps/backend/internal/github/oauth_flow.go` in
`GitHubOAuthClient.UserCanAccessInstallation`:

- request `GET /user/installations?per_page=100&page=<page>` through `apiRequest`, preserving the
  bearer token, media type, API-version header, response-size bound, and request context;
- decode each collection response's `total_count` and `installations[].id`, returning `true` as soon
  as the requested installation ID is found;
- continue until the response is exhausted, using the returned page length and positive
  `total_count` to avoid an unnecessary request, and return `false` only after successful
  exhaustion;
- return `GitHubAPIError` for any non-success status and a wrapped decode error for malformed JSON,
  so transport/protocol failures remain distinct from a verified absence.

Keep `AppInstallationService.userCanAccessInstallation` and its 250 ms, 500 ms, 1 second, and
2 second delays unchanged. It will now retry only successful list responses that do not contain the
target installation; client errors still stop immediately.

## Tests

- **What:** the OAuth client uses only the documented collection route and recognizes a matching
  installation.
  **File:** `apps/backend/internal/github/oauth_flow_test.go`.
  **How:** update `TestOAuthFlowGitHubClientExchangesRefreshesAndVerifiesUser` so its HTTP server
  explicitly rejects `/user/installations/{id}`, asserts `per_page=100` and `page=1`, and returns a
  realistic `{total_count, installations}` fixture from `/user/installations`.
- **What:** an installation on a later page is accessible and a target absent from every successful
  page is inaccessible.
  **File:** `apps/backend/internal/github/oauth_flow_test.go`.
  **How:** add a focused paginated server test with a full first page, a matching later page, and
  recorded page requests.
- **What:** GitHub HTTP or JSON failures are verification errors rather than an inaccessible result.
  **File:** `apps/backend/internal/github/oauth_flow_test.go`.
  **How:** cover a non-success collection response and a malformed successful collection response.
- **What:** a valid list omission remains eligible for the bounded callback retry and a later match
  can complete the installation.
  **File:** `apps/backend/internal/github/app_installation_service_test.go`.
  **How:** retain the existing false-then-true service test as the callback-level integration check;
  no production service change is required.

## Verification Results

- RED: the collection-route regression test failed because production requested
  `/user/installations/42` and returned `false, nil`.
- RED: the pagination regression test failed because the one-page implementation returned
  `false, nil` for the installation on page 2.
- GREEN: the targeted Go command in Task 01 passed four top-level tests, including both fail-closed
  subtests and the unchanged callback retry test.
- `git diff --check` passed.
- Go 1.26.0 ran from `/usr/local/go/bin/go`; `GOTMPDIR` used a temporary workspace directory because
  `/tmp` is mounted `noexec`. The temporary directory was removed after verification.

## Implementation Waves And Parallel Candidates

Sequential:

- [x] [task-01-correct-installation-association](task-01-correct-installation-association.md)

This task is not marked parallel-safe because the client and its tests form one TDD unit.

## Risks And Out Of Scope

- Pagination must terminate correctly when `total_count` is absent or inconsistent; a short page
  remains the authoritative exhaustion signal.
- Do not retain, persist, or log the temporary user token or any OAuth callback secret.
- Do not change App JWT/installation-token minting, webhook handling, callback routing, retry
  timings, frontend behavior, or public documentation. The existing public integration contract is
  unchanged; this repairs its GitHub API implementation.
