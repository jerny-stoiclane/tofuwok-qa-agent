---
name: apply-stale-sha
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Stale Plan: New push invalidates old plan, re-plan required before apply

Create PR, plan succeeds, push a new commit (plan is now stale). Tofuwok should re-plan automatically.
Verify the new plan supersedes the old one and apply works against the latest plan.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, HEAD_SHA_1

### Phase 2: Wait for Plan
Poll until plan completes for HEAD_SHA_1.
Verify: has_changes=true, lock acquired

### Phase 3: Push New Commit
Push another change to the same branch (makes the plan stale).
```bash
echo '# second commit' >> test/companies/bravo/snowflake/variables.tf
git add -A && git commit -m "test: second commit" && git push
```
Captures: HEAD_SHA_2

### Phase 4: Wait for Re-Plan
Tofuwok should detect the new push and dispatch a new plan for HEAD_SHA_2.
The old plan for HEAD_SHA_1 should be superseded.
Poll until new plan completes.

Assert:
- Two plan runs exist for this PR
- Newest plan is for HEAD_SHA_2

### Phase 5: Trigger Apply
Apply using auto-detect (tofuwok resolves the correct plan from PR HEAD):
```bash
bin/twk trigger --pr PR_NUMBER --action apply
```
Assert: apply dispatched successfully

### Phase 6: Wait for Apply + Merge
Wait for apply, then merge.

### Cleanup
Delete branch, release locks.
