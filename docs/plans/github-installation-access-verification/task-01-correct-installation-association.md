---
id: "01-correct-installation-association"
title: "Correct installation association lookup"
status: done
wave: 1
depends_on: []
plan: "plan.md"
spec: "../../specs/integrations/github-authentication.md"
---

# Task 01: Correct installation association lookup

## Intent

Make user-to-installation verification conform to GitHub's documented collection API so valid App
installation callbacks can complete without weakening the existing fail-closed association check.

## Acceptance

- `GitHubOAuthClient.UserCanAccessInstallation` requests only
  `GET /user/installations?per_page=100&page=<page>`, searches `installations[].id` across pages, and
  returns `true` for a match or `false` only after successful exhaustion.
- A non-success response, malformed collection, or request failure returns an error and cannot be
  mistaken for a verified absence.
- The HTTP-client test fails if `/user/installations/{installation_id}` is requested, covers a
  later-page match and an exhausted miss, and the existing callback-level false-then-true retry
  test remains green.

## Files Touched

- `apps/backend/internal/github/oauth_flow.go`
- `apps/backend/internal/github/oauth_flow_test.go`
- `docs/specs/integrations/github-authentication.md`
- `docs/plans/github-installation-access-verification/plan.md`
- `docs/plans/github-installation-access-verification/task-01-correct-installation-association.md`

## Dependencies

None.

## Inputs

- The amended association requirements in `docs/specs/integrations/github-authentication.md`.
- The backend conventions in `apps/backend/AGENTS.md`.
- GitHub's documented `GET /user/installations` response and pagination contract.
- The existing callback retry in `apps/backend/internal/github/app_installation_service.go`.

## Parallelism

`sequential`. The client behavior and its regression tests are one TDD unit.

## TDD And Verification

1. Mark this task `in_progress` and update `plan.md`.
2. Change/add the HTTP-server regression tests first. Run them before the production edit and record
   that they fail because the client requests `/user/installations/{id}` or does not decode and
   paginate the collection.
3. Implement the minimal collection-pagination fix.
4. Run:

   ```bash
   cd apps/backend && go test -tags fts5 ./internal/github -run 'TestOAuthFlowGitHubClientExchangesRefreshesAndVerifiesUser|TestGitHubOAuthClientUserCanAccessInstallation|TestAppInstallationCompleteAcceptsAutomaticOAuthInstallCallback' -count=1
   ```

5. Run `git diff --check`.
6. Reconcile the touched-file list, record exact outcomes below and in `plan.md`, mark both task and
   plan complete, and report any skipped check with its reason.

## Risks

- An inconsistent `total_count` must not hide a returned page that still contains the target or
  cause an unbounded page loop.
- Response bodies remain bounded by `maxGitHubAppResponseSize`, and diagnostics must not expose the
  bearer token.

## Output Contract

Report the summary, files changed, RED and GREEN test commands/results, `git diff --check` result,
blockers, residual risks, and synchronized task/plan status in this conversation.

## Results

- RED command:

  ```bash
  GOTMPDIR=<workspace-temp> /usr/local/go/bin/go test -v -tags fts5 ./internal/github -run '^TestOAuthFlowGitHubClientExchangesRefreshesAndVerifiesUser$' -count=1
  ```

  Failed as expected: the server rejected `/user/installations/42`, and
  `UserCanAccessInstallation()` returned `false, nil`.
- Pagination RED command:

  ```bash
  GOTMPDIR=<workspace-temp> /usr/local/go/bin/go test -v -tags fts5 ./internal/github -run '^TestGitHubOAuthClientUserCanAccessInstallationPaginates$' -count=1
  ```

  Failed as expected: the one-page implementation did not find the page-2 installation.
- GREEN command:

  ```bash
  GOTMPDIR=<workspace-temp> /usr/local/go/bin/go test -v -tags fts5 ./internal/github -run 'TestOAuthFlowGitHubClientExchangesRefreshesAndVerifiesUser|TestGitHubOAuthClientUserCanAccessInstallation|TestAppInstallationCompleteAcceptsAutomaticOAuthInstallCallback' -count=1
  ```

  Passed four top-level tests (six including subtests) in package `internal/github`.
- `git diff --check` passed.
- Go 1.26.0 was run from `/usr/local/go/bin/go`. `/tmp` was mounted `noexec`, so verification used
  an explicit temporary `GOTMPDIR` inside the workspace; that directory was removed afterward.
- Security/trust boundary: the temporary user token remains request-only and is not persisted or
  logged. Tests use only synthetic token text. External side effects: none.
