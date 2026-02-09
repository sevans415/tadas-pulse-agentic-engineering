---
name: perform-release
description: >
  Trigger a production release via GitHub Actions, monitor the ECS deployment,
  and verify the release is healthy. Use this skill when it's time to ship —
  after the PR is merged, CI is green, and staging has been verified. Also use
  it when asked to deploy, release, or ship to production.
user-invocable: true
---

# Perform Release

## Sequencing Checklist

- [ ] Verify prerequisites (`gh`, AWS credentials, MCP servers, deployment workflow)
- [ ] Pre-flight checks (confirm ref, verify CI is green, surface recent merged PRs)
- [ ] Trigger the deployment workflow
- [ ] Monitor the GitHub Actions workflow until it completes
- [ ] Monitor the ECS deployment (task health, service events, desired vs running count)
- [ ] Verify the release is live (health check, application logs, smoke tests, browser check)
- [ ] If verification fails, initiate rollback (see Rollback section below)
- [ ] Write and share the post-release report

Trigger a production deployment workflow, monitor it through the ECS rollout, and verify the release is live and healthy. When something goes wrong, use the available MCP servers to diagnose — don't just report that it failed.

## Prerequisites

**IMPORTANT: Check every prerequisite below BEFORE doing any work. If any check fails, stop immediately, tell the user which prerequisite is not met, and ask them to fix it. Do NOT proceed, improvise, or attempt workarounds.**

- The `gh` CLI must be installed and authenticated (`gh auth status` must succeed)
- AWS credentials must be configured (`aws sts get-caller-identity` must succeed)
- An **ECS MCP server** must be available — you need tools that can check ECS deployment status, task health, and service events. If no such tools are accessible, stop and tell the user.
- A **CloudWatch Logs MCP server** must be available — you need tools that can search and read CloudWatch log groups. If no such tools are accessible, stop and tell the user.
- A **CloudFormation / CDK MCP server** should be available for infrastructure troubleshooting. If not available, note it but continue — it's only needed if infrastructure issues arise.
- A production deployment workflow must exist — confirm with `gh workflow list` and identify the correct workflow. If none is found, stop and ask the user which workflow to use.
- The branch/ref to release must be identified. Default to `main` unless the user specifies otherwise.

## Steps

### 1. Pre-flight checks

Before triggering anything, confirm the release is safe to proceed:

```bash
# Confirm the ref to deploy
git log --oneline -5 main

# Confirm CI passed on the target ref
gh pr list --state merged --base main --limit 3 --json title,mergeCommit,statusCheckRollup
```

- If CI is not green on the target ref, stop and tell the user.
- If there are recent merged PRs the user may not be aware of, surface them and confirm they should be included in this release.

### 2. Trigger the deployment workflow

```bash
gh workflow run <deploy-production-workflow>.yml --ref main
```

Then locate the triggered run:

```bash
gh run list --workflow=<deploy-production-workflow>.yml --limit=1 --json databaseId,status,conclusion,headBranch,createdAt
```

Capture the run ID — you'll need it throughout.

### 3. Monitor the GitHub Actions workflow

Poll until the GHA workflow completes:

```bash
gh run watch <run-id>
```

If the workflow fails:
- Read the logs: `gh run view <run-id> --log-failed`
- If the failure is in a CDK deploy step, use the CloudFormation / CDK MCP server to inspect stack events and identify the root cause.
- Report what failed and why. Do not re-trigger without user approval.

### 4. Monitor the ECS deployment

Once the GHA workflow succeeds, the ECS deployment is in progress. Use the ECS MCP server to monitor it:

- Check deployment status — confirm the new task definition is rolling out and the primary deployment is progressing.
- Watch for task failures — if tasks are failing to start, inspect the stopped task reasons (OOM, image pull failures, health check failures, etc.).
- Monitor service events — look for steady-state messages confirming the new tasks are healthy.
- Check that the desired count matches the running count once rollout completes.

If the ECS deployment stalls or tasks keep failing:
- Use the ECS MCP server to get stopped task details and failure reasons.
- Use the CloudWatch Logs MCP server to pull recent application logs from the failing tasks.
- If the failure is infrastructure-related (networking, IAM, capacity), use the CloudFormation / CDK MCP server to inspect the CDK-managed resources.
- Report what went wrong with specifics. Do not guess — use the tools.

### 5. Verify the release is live

Once ECS shows a stable deployment:

- **Health check**: hit the production health endpoint and confirm a 200:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" https://<production-url>/health
  ```
- **Application logs**: use the CloudWatch Logs MCP server to check the last few minutes of logs for errors, exceptions, or unexpected patterns. Compare error rates to pre-deploy baseline if possible.
- **Smoke test key endpoints**: use `curl` to hit 2-3 critical API endpoints and confirm they respond correctly.
- **Browser check** (if a Playwright MCP server is available): navigate to the production URL and confirm the main page loads and critical UI flows work.

If any verification step fails:
- Do NOT ignore it. Investigate using the MCP servers.
- If the issue is a regression from this release, tell the user immediately and recommend a rollback.

### 6. Post-release report

Summarize the release:

```
## Production Release

**Ref**: <commit or tag>
**Workflow run**: <link to gh run>
**ECS service**: <cluster/service name>

### Deployment
- GHA workflow: passed/failed
- ECS rollout: completed/failed — <task def revision>
- Time to stable: <approximate duration>

### Verification
- Health check: <status>
- Application logs: <clean / errors found>
- Smoke tests: <what was checked and result>

### Issues encountered
- <issue>: <what happened, how it was resolved or escalated>
```

---

## Rollback

If the user asks to roll back, or if verification reveals a critical regression:

1. Identify the previous stable task definition using the ECS MCP server.
2. Trigger the deployment workflow for the previous known-good ref, or ask the user how rollbacks are handled in their pipeline.
3. Monitor the rollback deployment using the same steps above (steps 3-5).
4. Confirm the rollback is healthy before considering it done.

Do NOT perform a rollback without explicit user approval unless they have pre-authorized it.
