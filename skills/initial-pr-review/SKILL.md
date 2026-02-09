---
name: initial-pr-review
description: >
  Run a structured first-pass code review on a PR. Use this skill when asked
  to review a PR, when opening a PR that should be sanity-checked before
  requesting human review, or when picking up someone else's PR to understand
  what changed and whether it's ready to merge.
user-invocable: true
---

# Initial PR Review

## Sequencing Checklist

- [ ] Verify prerequisites (`gh auth status`, PR is accessible)
- [ ] Gather PR context (`gh pr view`, `gh pr diff`)
- [ ] Review: Correctness
- [ ] Review: Security
- [ ] Review: Breaking changes
- [ ] Review: Tests
- [ ] Review: Naming and clarity
- [ ] Review: Complexity and scope
- [ ] Review: Operational impact
- [ ] Write the review (Summary, Issues, Suggestions, Questions)
- [ ] Post the review to GitHub (only if explicitly requested)

Perform a structured first-pass review of a pull request, checking for the things the team has agreed matter most. This is not a nitpick pass — it's a "catch real problems early" pass.

## Prerequisites

**IMPORTANT: Check every prerequisite below BEFORE doing any work. If any check fails, stop immediately, tell the user which prerequisite is not met, and ask them to fix it. Do NOT proceed, improvise, or attempt workarounds.**

- The `gh` CLI must be installed and authenticated (`gh auth status` must succeed)
- You must be given a PR number, URL, or be on a branch with an open PR (`gh pr view` must succeed for the target PR)

## Steps

### 1. Gather context

```bash
gh pr view <pr> --json title,body,baseRefName,headRefName,files,additions,deletions
gh pr diff <pr>
```

Read the PR title, description, and full diff. Understand what the PR is trying to accomplish before reviewing any code.

### 2. Run the checklist

Work through each section below. For each item, note whether it passes, has concerns, or is not applicable. Only flag things that are genuine problems — do not manufacture issues.

---

## Review Checklist

<!-- ===========================================================
     ADD NEW CHECKLIST SECTIONS HERE

     Each section should follow this format:

     ### Section Name
     - **What to check**: one-line description
     - **How to check**: concrete steps or commands
     - **Common mistakes**: patterns to watch for

     Keep sections focused on one concern. A section with more
     than 4-5 bullets is probably two sections.
     =========================================================== -->

### Correctness

- Does the code do what the PR description says it does?
- Are there logic errors, off-by-one mistakes, or unhandled edge cases?
- Are error paths handled? Will failures surface clearly or fail silently?

### Security

- Is user input validated/sanitized before use?
- Are there hardcoded secrets, tokens, or credentials?
- Are new dependencies from trusted sources and pinned to specific versions?
- Does the change introduce any OWASP Top 10 risks (injection, broken auth, XSS, etc.)?

### Breaking changes

- Does this change modify a public API, database schema, config format, or wire protocol?
- If so, is it backward-compatible or is there a migration path?
- Are existing callers/consumers updated?

### Tests

- Are new code paths covered by tests?
- Do existing tests still make sense after this change, or do some need updating?
- Are tests testing behavior (what) rather than implementation (how)?

### Naming and clarity

- Are new functions, variables, and files named so that a reader unfamiliar with the PR can follow the code?
- Is there anything confusing enough that it needs a comment, but doesn't have one?

### Complexity and scope

- Is the PR doing more than one thing? Should it be split?
- Is there unnecessary abstraction, over-engineering, or speculative generality?
- Could any part of this be simpler while still being correct?

### Operational impact

- Are there new environment variables, feature flags, or config changes required?
- Could this change affect performance, memory usage, or latency at scale?
- Are there new log lines, metrics, or alerts that should accompany this change?

---

## 3. Write the review

Produce a review comment with this structure:

```
## Summary
One or two sentences on what this PR does and the overall impression.

## Issues
Items that should be fixed before merging. Reference specific files and lines.

## Suggestions
Non-blocking improvements the author could consider.

## Questions
Anything that wasn't clear from the code or description alone.
```

If the PR looks good, say so plainly. Do not pad the review with filler.

## 4. Post the review (if requested)

Only post to GitHub if explicitly asked. Use:

```bash
gh pr review <pr> --comment --body "$(cat <<'EOF'
<review content here>
EOF
)"
```

Use `--approve` or `--request-changes` in place of `--comment` when appropriate.
