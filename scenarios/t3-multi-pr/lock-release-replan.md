---
name: lock-release-replan
tier: t3
timeout: 900
---

# Lock Release → Replan: Full Contention Cycle

Tests the complete lock contention cycle:
1. PR1 plans dir A → lock acquired
2. PR2 plans dir A → lock conflict
3. PR1 merges → apply → lock released
4. PR2 re-plans (new push or branch update) → lock acquired
5. PR2 merges → apply → lock released

This is the core coordination flow that prevents stale plan merges.

## Phases

### Phase 1: Create PR1
skill: create-test-branch
  RUN_ID: $RUN_ID-pr1
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] lock-release-replan PR1"

Captures: PR1_NUMBER, PR1_SHA

### Phase 2: Wait for PR1 Plan + Verify Lock
skill: wait-for-plans (PR1, bravo/snowflake)
Assert: lock held by PR1

### Phase 3: Create PR2 (Same Dir)
skill: create-test-branch
  RUN_ID: $RUN_ID-pr2
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] lock-release-replan PR2"

Captures: PR2_NUMBER, PR2_SHA

### Phase 4: Wait for PR2 Plan + Verify Conflict
skill: wait-for-plans (PR2, bravo/snowflake)
Assert: lock still held by PR1, PR2 sees conflict

### Phase 5: Merge PR1 → Apply
skill: merge-pr (PR1)
skill: wait-for-applies (PR1, bravo/snowflake)
Assert: apply succeeded, lock released (or applied=true then released on merge)

### Phase 6: Retrigger PR2 Plan
After PR1's lock is released, PR2 needs a fresh plan.
Push an empty commit to PR2 to retrigger:
```bash
cd /Users/jernz/stoiclane/tf-orchestrator-gha
git checkout test-qa/$RUN_ID-pr2
git commit --allow-empty -m "ci: retrigger after PR1 apply"
git push
```

### Phase 7: Wait for PR2 Re-Plan + Verify Lock Acquired
skill: wait-for-plans (PR2, bravo/snowflake)
Assert: lock now held by PR2 (not PR1)

### Phase 8: Merge PR2 → Apply
skill: merge-pr (PR2)
skill: wait-for-applies (PR2, bravo/snowflake)
Assert: apply succeeded, lock released, no orphan locks

### Cleanup
skill: cleanup
- Both PRs already merged — just clean branches and locks
