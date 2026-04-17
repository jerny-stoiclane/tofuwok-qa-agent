---
name: cancel-during-plan
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Cancel During Plan

Create PR, plan starts, cancel it via tofuwok API while plan is in progress. Verify lock released and run marked cancelled.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan to Start
Poll tofuwok until a plan run exists with status=running (not success/failure yet).
Timeout: 60s

### Phase 3: Cancel the Run
```bash
bin/twk cancel --run-id RUN_ID
```

### Phase 4: Verify Cancel
Assert via tofuwok API:
- Run status changed to "cancelled"
- Lock released (no lock held for this dir)
- GHA workflow either stopped or completed with cancellation

### Cleanup
Delete branch, release any remaining locks.
