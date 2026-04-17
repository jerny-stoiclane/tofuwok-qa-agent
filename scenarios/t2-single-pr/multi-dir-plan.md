---
name: multi-dir-plan
tier: t2
timeout: 600
dirs:
  - test/companies/_global/networking
  - test/companies/alpha/aws/us-east-1/platform
  - test/companies/alpha/snowflake
  - test/companies/bravo/aws/us-east-1/platform
  - test/companies/bravo/snowflake
  - test/companies/charlie/snowflake
---

# Multi Dir: Plan All Companies

Tests: create PR touching all 6 dirs → tofuwok detects all → dispatches 6 plans in parallel → all succeed → 6 locks acquired.

## Phases

### Phase 1: Create Branch + PR
skill: create-test-branch
  dirs: [all 6 dirs listed above]
skill: create-test-pr
  title: "[qa] multi-dir-plan"

Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for All Plans via Tofuwok API
Poll tofuwok runs API until ALL 6 dirs have terminal plan runs for this PR.
```
GET /api/v1/runs/{owner}/{repo} → filter pr_number=PR_NUMBER, run_type=plan
```
Timeout: 300s, poll every 15s. Done when 6 runs with terminal status exist.

### Phase 3: Verify Plan Results via Tofuwok API

For each of the 6 dirs, assert via tofuwok API:
- Run exists with status=success, run_type=plan (runs API)
- Lock exists with pr_number == PR_NUMBER, applied == false (locks API)
- Commit status tofuwok/plan/{dir} == success on HEAD_SHA (GitHub API — set by tofuwok)

Assert PR comment (GitHub API — posted by tofuwok):
- Comment contains plan results for all 6 dirs

### Cleanup
skill: cleanup
