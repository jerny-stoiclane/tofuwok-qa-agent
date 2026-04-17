---
name: apply-stale-sha
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Stale SHA: Apply rejected when plan is outdated

Create PR, plan succeeds, push a new commit (plan is now stale), try to apply with the old SHA. Apply should be rejected.

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

### Phase 5: Trigger Apply with Old SHA
Try to apply using HEAD_SHA_1 (the stale SHA):
```bash
bin/twk trigger --pr PR_NUMBER --action apply --sha HEAD_SHA_1 --dir test/companies/bravo/snowflake
```
Assert: apply is rejected (400 "no plan" or "stale plan")

### Phase 6: Trigger Apply with Current SHA
Apply using HEAD_SHA_2 (the current SHA):
```bash
bin/twk trigger --pr PR_NUMBER --action apply --dir test/companies/bravo/snowflake --sha HEAD_SHA_2
```
Assert: apply dispatched successfully

### Phase 7: Wait for Apply + Merge
Wait for apply, then merge.

### Cleanup
Delete branch, release locks.
