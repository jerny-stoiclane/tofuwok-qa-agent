---
name: push-after-apply
tier: t2
timeout: 600
dirs:
  - test/companies/bravo/snowflake
---

# Push After Apply: New commit invalidates applied state

Plan succeeds, apply succeeds, all checks green. Then a new commit is pushed before merge. The applied state no longer matches HEAD. Tofuwok should invalidate the apply and require re-plan + re-apply.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, SHA_1

### Phase 2: Wait for Plan
Poll tofuwok until plan completes. Verify has_changes=true, lock acquired.

### Phase 3: Trigger Apply
```bash
bin/twk trigger --pr PR_NUMBER --action apply
```
Wait for apply to complete. Verify success, lock applied=true.

### Phase 4: Push Another Commit
Push a new change to the same branch (e.g., add another resource or modify a variable):
```bash
echo 'resource "null_resource" "forgot_this" { triggers = { v = "oops" } }' >> test/companies/bravo/snowflake/main.tf
git add -A && git commit -m "test: push after apply" && git push
```
Captures: SHA_2

### Phase 5: Verify Invalidation
Assert via tofuwok API:
- The new push (synchronize event) should trigger a new plan for SHA_2
- The old apply for SHA_1 should be invalidated — tofuwok should NOT consider the PR as "applied"
- The lock should be re-acquired for the new plan (or remain held, but applied flag reset to false)
- GitHub check runs for apply should be reset/invalidated (not still showing the old apply as green)

### Phase 6: Verify Merge Blocked
Assert:
- PR should NOT be mergeable with a stale apply
- The apply check from SHA_1 should not satisfy the requirement for SHA_2
- Tofuwok should require re-apply on the new commit

### Phase 7: Re-Apply + Merge
Trigger apply for the new plan, wait for completion, then merge.

### Cleanup
Delete branch, release locks.
