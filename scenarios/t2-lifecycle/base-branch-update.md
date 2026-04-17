---
name: base-branch-update
tier: t2
timeout: 600
dirs:
  - test/companies/bravo/snowflake
---

# Base Branch Update: Stale plan after main changes

PR1 plans, PR2 merges first (changes main), PR1's plan is now stale. Tests whether tofuwok handles this correctly.

## Phases

### Phase 1: Create PR1
Add null_resource "pr1_test", push, create PR.
Captures: PR1_NUMBER, PR1_SHA

### Phase 2: Wait for PR1 Plan
Poll until plan completes. Verify lock acquired.

### Phase 3: Create PR2 (Different Change, Same Dir)
Checkout main, create new branch, add null_resource "pr2_test" to same dir.
Captures: PR2_NUMBER, PR2_SHA

### Phase 4: Wait for PR2 Plan
PR2 should see lock conflict (PR1 holds the lock).

### Phase 5: Apply + Merge PR2 First
This requires PR1 to release its lock first, OR force-release.
If locked by PR1: force-release, then trigger PR2 apply.
Apply PR2, merge PR2. Main now has PR2's changes.

### Phase 6: Verify PR1 Plan is Stale
PR1's plan was generated before PR2's apply changed state.
Check if tofuwok detects this and requires re-plan.

### Phase 7: Re-Plan PR1
Push empty commit to PR1 or trigger re-plan.
Wait for new plan that accounts for PR2's changes.

### Phase 8: Apply + Merge PR1

### Cleanup
Delete branches, release locks.
