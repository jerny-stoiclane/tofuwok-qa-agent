---
name: destroy-detection
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Destroy Detection: Plan with resource removal

First apply a resource, then create a PR that removes it. Plan should show resources_to_destroy > 0.

NOTE: This test depends on state from a previous apply. The first phase sets up state.

## Phases

### Phase 1: Setup — Create and Apply a Resource
Create PR with a null_resource, plan, apply, merge. This puts the resource in state.

### Phase 2: Create Removal PR
Create new branch that REMOVES the null_resource from main.tf.
Create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 3: Wait for Plan
Poll until plan completes.

### Phase 4: Verify Destroy Plan
Assert via tofuwok API:
- Run exists with status=success, has_changes=true
- resources_to_destroy > 0
- resources_to_add == 0

### Phase 5: Apply + Merge
Trigger apply, wait, merge.

### Cleanup
Delete branch, release locks.
