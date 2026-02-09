---
name: test-in-staging
description: >
  Deploy the current branch to staging and verify it works end-to-end. Use
  this skill after CI passes when you need to confirm changes actually work
  in a real environment before merging — especially for user-facing changes,
  API modifications, or anything that interacts with external services. Do
  not consider a PR "ready to merge" for non-trivial changes without a
  staging check.
user-invocable: true
---

# Test in Staging

## Sequencing Checklist

- [ ] Verify prerequisites (`gh`, staging workflow, browser automation MCP, staging URL)
- [ ] Deploy to staging and wait for the workflow to complete
- [ ] Verify: Smoke test — app is up (health endpoint returns 200)
- [ ] Verify: Changed endpoints and pages respond correctly
- [ ] Verify: Auth and permissions (if applicable)
- [ ] Verify: Error handling (malformed inputs return clear errors)
- [ ] Verify: Data integrity (if writes are involved)
- [ ] Verify: Cross-feature regression (adjacent flows still work)
- [ ] Report results (passed, fixed during verification, not applicable)
- [ ] Clean up staging environment (if needed)

Deploy the current branch to staging and run through verification steps to confirm the changes work in a real environment. When a check fails, fix the issue, redeploy, and re-verify — don't just report the failure.

## Prerequisites

**IMPORTANT: Check every prerequisite below BEFORE doing any work. If any check fails, stop immediately, tell the user which prerequisite is not met, and ask them to fix it. Do NOT proceed, improvise, or attempt workarounds.**

- The `gh` CLI must be installed and authenticated (`gh auth status` must succeed)
- A staging deployment workflow must exist — confirm with `gh workflow list` and identify the correct workflow name. If none is found, stop and ask the user which workflow to use.
- For browser-based verification: a browser automation MCP server (e.g. Playwright) must be available. If no browser automation tools are accessible, tell the user and skip only the browser-based checks (still run all `curl`-based checks).
- The staging environment URL must be known. If not provided and not discoverable from the repo, stop and ask the user for it.

## Steps

### 1. Deploy to staging

Trigger the staging deployment workflow for the current branch:

```bash
gh workflow run <deploy-staging-workflow>.yml --ref $(git branch --show-current)
```

Then wait for the deployment to complete:

```bash
gh run list --workflow=<deploy-staging-workflow>.yml --branch=$(git branch --show-current) --limit=1 --json status,conclusion,databaseId
```

Poll until the run completes. If the deploy itself fails, inspect the logs with `gh run view <run-id> --log-failed` and fix before proceeding.

### 2. Run the verification checklist

Work through each section below. For each item, actually perform the check — don't just reason about whether it should work. Use `curl` for API checks and a browser automation MCP server for anything that requires a browser.

**When a check fails**: fix the underlying issue, commit, push, redeploy to staging, and re-run the failed check. Repeat until it passes. Do not skip ahead.

---

## Verification Checklist

<!-- ===========================================================
     ADD NEW CHECKLIST SECTIONS HERE

     Each section should follow this format:

     ### Section Name
     - **What to verify**: one-line description
     - **How to verify**: curl command, Playwright steps, or other
     - **Fix-and-redeploy if**: conditions that mean this failed

     Keep sections focused on one concern.
     =========================================================== -->

### Smoke test — app is up

- Hit the health/root endpoint and confirm a 200:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" https://<staging-url>/health
  ```
- If this fails, nothing else will work. Check deploy logs first.

### Changed endpoints and pages

- For each API endpoint or page touched by the PR, make a real request and verify the response.
- **API changes**: use `curl` to call the endpoint with representative inputs and check the response body and status code.
  ```bash
  curl -s https://<staging-url>/api/example | jq .
  ```
- **UI changes**: use the browser automation MCP server to navigate to the page, interact with the changed elements, and confirm they render and behave correctly.
- Compare actual behavior against what the PR description says should happen.

### Auth and permissions

- If the change touches authentication or authorization, verify:
  - Unauthenticated requests are rejected where expected
  - Authenticated requests with correct roles succeed
  - Authenticated requests with wrong roles are denied
- Use `curl` with and without auth headers to confirm.

### Error handling

- Send malformed or missing inputs to changed endpoints and confirm the response is a clear error (not a 500 or stack trace).
  ```bash
  curl -s -w "\n%{http_code}" https://<staging-url>/api/example -d '{}'
  ```

### Data integrity

- If the change writes to a database or external service, verify:
  - Data is written correctly (query staging DB or check the service)
  - Invalid writes are rejected
  - No orphaned or duplicate records are created

### Cross-feature regression

- Briefly check that the main flows adjacent to the changed area still work.
- For UI: use the browser automation MCP server to walk through one happy-path flow that touches the changed area.
- For API: hit 2-3 related endpoints that weren't changed and confirm they still return expected results.

---

## 3. Report results

Summarize what was verified and the outcome:

```
## Staging Verification

**Environment**: <staging-url>
**Branch**: <branch-name>
**Deploy run**: <link to gh run>

### Passed
- <check>: <what was confirmed>

### Fixed during verification
- <check>: <what failed, what was fixed, commit ref>

### Not applicable
- <check>: <why it doesn't apply to this PR>
```

## 4. Clean up (if needed)

If your staging environment requires cleanup (reverting test data, resetting state), do that before finishing.
