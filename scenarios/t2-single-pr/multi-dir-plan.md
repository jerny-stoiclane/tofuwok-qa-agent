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

### Phase 2: Wait for Affected Detection
Wait for affected.yaml workflow on this branch.
Timeout: 90s

### Phase 3: Wait for All Plans
skill: wait-for-plans
  pr_number: PR_NUMBER
  expected_dirs: [all 6]
Timeout: 300s (6 parallel plans)

### Phase 4: Verify Plan Results

For each of the 6 dirs, assert:
- tofuwok run exists with status=success, run_type=plan
- Lock exists with pr_number == PR_NUMBER, applied == false
- Commit status tofuwok/plan/{dir} == success on HEAD_SHA

Assert PR comment:
- Comment contains plan results for all 6 dirs

### Cleanup
skill: cleanup
