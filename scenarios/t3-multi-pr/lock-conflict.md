---
name: lock-conflict
tier: t3
timeout: 600
---

# Lock Conflict: Two PRs, Same Dir

Tests: PR1 plans dir A → lock acquired → PR2 plans dir A → lock conflict (409) → PR2 merge blocked.

## Phases

### Phase 1: Create PR1
skill: create-test-branch
  RUN_ID: $RUN_ID-pr1
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] lock-conflict PR1 (lock holder)"

Captures: PR1_NUMBER, PR1_SHA

### Phase 2: Wait for PR1 Plan
skill: wait-for-plans
  pr_number: PR1_NUMBER
  expected_dirs: [test/companies/bravo/snowflake]

Assert:
- PR1 plan succeeded
- Lock acquired by PR1

### Phase 3: Create PR2 (Same Dir)
skill: create-test-branch
  RUN_ID: $RUN_ID-pr2
  dirs: [test/companies/bravo/snowflake]

NOTE: Must checkout from main, not from PR1's branch.

skill: create-test-pr
  title: "[qa] lock-conflict PR2 (blocked)"

Captures: PR2_NUMBER, PR2_SHA

### Phase 4: Wait for PR2 Plan
skill: wait-for-plans
  pr_number: PR2_NUMBER
  expected_dirs: [test/companies/bravo/snowflake]

The plan itself should succeed (terraform runs fine), but the lock should show conflict.

### Phase 5: Verify Lock Conflict

Assert lock state:
- Lock still held by PR1 (not PR2)
- lock.pr_number == PR1_NUMBER

Assert PR2 status:
- PR2 should have a lock conflict indicator
  Either: commit status "tofuwok/lock/..." = pending
  Or: tofuwok run shows lock_conflict in status_reason

Assert PR2 comment:
- PR2 comment should mention lock conflict or "locked by PR #PR1_NUMBER"

### Phase 6: Verify PR2 Cannot Merge Cleanly
- PR2's required status checks should be failing/pending due to lock
- (This depends on branch protection being configured — may be a soft assertion)

### Cleanup
skill: cleanup
- Close PR2 (it was never merged)
- Close PR1 (it was never merged)
- Release all locks
- Delete both test branches
