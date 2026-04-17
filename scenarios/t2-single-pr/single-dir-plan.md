---
name: single-dir-plan
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Single Dir: Plan Lifecycle

Tests: create PR with 1 dir change → tofuwok detects affected → dispatches plan → plan succeeds → lock acquired → commit status set.

## Phases

### Phase 1: Create Branch + PR
skill: create-test-branch
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] single-dir-plan"

Captures: PR_NUMBER, HEAD_SHA, BRANCH

### Phase 2: Wait for Plan via Tofuwok API
Poll tofuwok runs API until a plan run exists for this PR + dir with terminal status.
```
GET /api/v1/runs/{owner}/{repo} → filter pr_number=PR_NUMBER, run_type=plan, dir=test/companies/bravo/snowflake
```
Timeout: 180s, poll every 15s

### Phase 3: Verify Plan Results via Tofuwok API

Assert plan run (tofuwok runs API):
- Run exists with status=success, run_type=plan, dir=test/companies/bravo/snowflake
- run.has_changes == true

Assert lock (tofuwok locks API):
- `GET /api/v1/locks/{owner}/{repo}` shows lock for test/companies/bravo/snowflake
- lock.pr_number == PR_NUMBER
- lock.applied == false

Assert commit status (GitHub API — this is set BY tofuwok, we verify it landed):
- tofuwok/plan/test/companies/bravo/snowflake == success on HEAD_SHA

Assert PR comment (GitHub API — posted BY tofuwok):
- PR has a comment with plan output

### Cleanup
skill: cleanup
