---
name: plan-apply-merge
tier: t2
timeout: 600
dirs:
  - test/companies/bravo/snowflake
---

# Full Cycle: Plan → Merge → Apply

Tests the complete lifecycle: PR created → plan succeeds → lock acquired → PR merged → apply dispatched → apply succeeds → lock released.

## Phases

### Phase 1: Create Branch + PR
skill: create-test-branch
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] plan-apply-merge"

Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan via Tofuwok API
Poll tofuwok runs API for plan run with terminal status.
Timeout: 180s

Verify via tofuwok API:
- Run status=success, has_changes=true (runs API)
- Lock acquired, pr_number=PR_NUMBER, applied=false (locks API)
- Commit status success (GitHub API — set by tofuwok)

### Phase 3: Merge PR
```bash
gh pr merge PR_NUMBER --repo TARGET_REPO --merge
```
Captures: MERGE_SHA

### Phase 4: Wait for Apply via Tofuwok API
Poll tofuwok runs API for apply run with terminal status.
```
GET /api/v1/runs/{owner}/{repo} → filter pr_number=PR_NUMBER, run_type=apply
```
Timeout: 300s

### Phase 5: Verify Apply Results via Tofuwok API

Assert apply run (runs API):
- Run exists with status=success, run_type=apply, dir=test/companies/bravo/snowflake

Assert lock (locks API):
- No lock exists for test/companies/bravo/snowflake (released after apply+merge)
  OR lock exists with applied=true (depends on lock policy timing)

Assert commit status (GitHub API — set by tofuwok):
- tofuwok/apply/test/companies/bravo/snowflake == success

Assert PR comment (GitHub API — posted by tofuwok):
- Apply result posted to the merged PR

### Cleanup
skill: cleanup
- No PRs to close (already merged)
- Clean any remaining locks
- Delete test branch (if not auto-deleted by merge)
