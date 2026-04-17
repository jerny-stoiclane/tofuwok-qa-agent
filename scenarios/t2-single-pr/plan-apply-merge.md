---
name: plan-apply-merge
tier: t2
timeout: 600
dirs:
  - test/companies/bravo/snowflake
---

# Full Cycle: Plan → Apply → Merge

Tests the complete lifecycle: PR created → plan succeeds → lock acquired → apply triggered → apply succeeds → lock marked applied → PR merged → lock released.

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

### Phase 3: Trigger Apply via Tofuwok API
```
POST /api/v1/trigger with run_type=apply for test/companies/bravo/snowflake
```

### Phase 4: Wait for Apply via Tofuwok API
Poll tofuwok runs API for apply run with terminal status.
Timeout: 300s

### Phase 5: Verify Apply Results

Assert apply run (tofuwok runs API):
- Run exists with status=success, run_type=apply, dir=test/companies/bravo/snowflake

Assert lock (tofuwok locks API) — **check BEFORE merging**:
- Lock exists with applied=true

Assert apply status via **GitHub Check Runs API** (not commit statuses):
```bash
gh api repos/{owner}/{repo}/commits/{sha}/check-runs \
  --jq '.check_runs[] | select(.name | contains("apply")) | {name, conclusion}'
```
- Check run `tofuwok/apply` exists with conclusion=success
- Check run `tofuwok/apply/test/companies/bravo/snowflake` exists with conclusion=success

Note: tofuwok posts **plan** results as commit statuses (`tofuwok/plan`) and **apply** results
as GitHub Check Runs (`tofuwok/apply`, `tofuwok/apply/{dir}`).

### Phase 6: Merge PR
Apply succeeded — merge to land code on main.
```bash
gh pr merge PR_NUMBER --repo TARGET_REPO --merge
```
Captures: MERGE_SHA

### Phase 7: Verify Lock Released After Merge
Assert lock (tofuwok locks API):
- No lock exists for test/companies/bravo/snowflake (released on merge event)

### Cleanup
skill: cleanup
- No PRs to close (already merged)
- Clean any remaining locks
- Delete test branch (if not auto-deleted by merge)
