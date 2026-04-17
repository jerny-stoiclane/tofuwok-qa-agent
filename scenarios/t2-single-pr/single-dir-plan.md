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

### Phase 2: Wait for Affected Detection
skill: wait for affected.yaml workflow to complete on this branch
Timeout: 90s

### Phase 3: Wait for Plan
skill: wait-for-plans
  pr_number: PR_NUMBER
  expected_dirs: [test/companies/bravo/snowflake]
Timeout: 180s

### Phase 4: Verify Plan Results

Assert plan run:
- tofuwok run exists with status=success, run_type=plan, dir=test/companies/bravo/snowflake
- run.has_changes == true (we added a comment to variables.tf)

Assert lock:
- Lock exists for test/companies/bravo/snowflake
- lock.pr_number == PR_NUMBER
- lock.applied == false
- lock.lock_policy == "strict"

Assert commit status:
- tofuwok/plan/test/companies/bravo/snowflake == success on HEAD_SHA

Assert PR comment:
- PR has a comment from tofuwok with plan output

### Cleanup
skill: cleanup
