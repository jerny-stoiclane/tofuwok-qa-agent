---
name: force-release
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Force Release: Admin unlock

Create PR, plan acquires lock, force-release the lock via API. Verify lock is gone and PR can be re-planned.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER

### Phase 2: Wait for Plan + Verify Lock
Poll until plan completes. Verify lock acquired.

### Phase 3: Force Release
```bash
curl -sf -X DELETE -H "Authorization: Bearer $TOFUWOK_TOKEN" \
  "$TOFUWOK_API/api/v1/locks/$OWNER/$REPO/force?dir=test/companies/bravo/snowflake&workspace=default"
```
Or via bin/twk if supported.

### Phase 4: Verify Lock Released
Assert: no lock exists for this dir.

### Phase 5: Re-Plan
Push a trivial commit to retrigger plan.
Verify: new plan runs, new lock acquired.

### Phase 6: Apply + Merge
Trigger apply, merge.

### Cleanup
Delete branch.
