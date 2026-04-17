---
name: cancel-during-apply
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Cancel During Apply: State safety

Trigger apply, then cancel via tofuwok API while apply is running. This is the dangerous case — terraform apply mid-execution should NOT be hard-killed (corrupts state).

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR, wait for plan.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Trigger Apply
```bash
bin/twk trigger --pr PR_NUMBER --action apply
```

### Phase 3: Wait for Apply to Start
Poll tofuwok until apply run exists with status=running.
Timeout: 60s

### Phase 4: Cancel the Run
```bash
bin/twk cancel --run-id RUN_ID
```

### Phase 5: Verify Safe Cancellation
Assert via tofuwok API:
- Run status is "cancelled" or "success" (apply may have finished before cancel took effect)
- If cancelled: terraform was NOT killed mid-resource (soft cancel — finish current resource)
- Lock state is consistent (released if cancelled, applied=true if completed)
- No state corruption (next plan on this dir should work cleanly)

### Phase 6: Verify No State Corruption
Push empty commit to retrigger plan. Verify plan runs cleanly without state errors.

### Cleanup
Close or merge PR depending on what happened. Delete branch, release locks.
