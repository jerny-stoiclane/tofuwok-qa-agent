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

### Phase 2: Wait for Plan
skill: wait-for-plans
  pr_number: PR_NUMBER
  expected_dirs: [test/companies/bravo/snowflake]

Assert after plan:
- Run status=success, has_changes=true
- Lock acquired, pr_number=PR_NUMBER, applied=false
- Commit status success

### Phase 3: Merge PR
skill: merge-pr
  pr_number: PR_NUMBER

Captures: MERGE_SHA

### Phase 4: Wait for Apply
skill: wait-for-applies
  pr_number: PR_NUMBER
  expected_dirs: [test/companies/bravo/snowflake]
Timeout: 300s

### Phase 5: Verify Apply Results

Assert apply run:
- tofuwok run exists with status=success, run_type=apply, dir=test/companies/bravo/snowflake

Assert lock released:
- No lock exists for test/companies/bravo/snowflake (released after apply+merge)
  OR lock exists with applied=true (depends on lock policy timing)

Assert commit status:
- tofuwok/apply/test/companies/bravo/snowflake == success

Assert PR comment:
- Apply result posted to the merged PR

### Cleanup
skill: cleanup
- No PRs to close (already merged)
- Clean any remaining locks
- Delete test branch (if not auto-deleted by merge)
