---
name: pr-synchronize
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# PR Synchronize: New push triggers re-plan

Create PR, plan succeeds, push a new commit. Tofuwok should detect the synchronize event and dispatch a new plan.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, SHA_1

### Phase 2: Wait for First Plan
Poll until plan completes for SHA_1.

### Phase 3: Push New Commit
```bash
echo 'resource "null_resource" "sync_test" { triggers = { v = "2" } }' >> test/companies/bravo/snowflake/main.tf
git add -A && git commit -m "test: synchronize" && git push
```
Captures: SHA_2

### Phase 4: Wait for Second Plan
Poll until a NEW plan run exists for SHA_2 (not SHA_1).
Timeout: 180s

### Phase 5: Verify
Assert:
- New plan run exists for SHA_2 with status=success
- Lock still held by this PR (same pr_number)
- Old plan for SHA_1 may be superseded

### Phase 6: Apply + Merge
Trigger apply for latest plan, merge.

### Cleanup
Delete branch, release locks.
