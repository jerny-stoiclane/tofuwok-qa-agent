---
name: acquire-release
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Lock Acquire + Release: Basic lifecycle

Verify the full lock lifecycle: plan acquires → held during execution → released after apply+merge.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll until plan completes.

### Phase 3: Verify Lock Acquired
Assert via tofuwok locks API:
- Lock exists for test/companies/bravo/snowflake
- lock.pr_number == PR_NUMBER
- lock.applied == false
- lock.locked_by is set

### Phase 4: Trigger Apply
```bash
bin/twk trigger --pr PR_NUMBER --action apply
```
Wait for apply to complete.

### Phase 5: Verify Lock Applied
Assert:
- Lock still exists (not yet released — need merge)
- lock.applied == true

### Phase 6: Merge PR
```bash
gh pr merge PR_NUMBER --repo TARGET_REPO --merge
```

### Phase 7: Verify Lock Released
Wait up to 30s, then assert:
- No lock exists for this dir (released on merge)

### Cleanup
Delete branch.
