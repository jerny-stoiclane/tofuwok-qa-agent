---
name: drift-detection
tier: t2
timeout: 600
dirs:
  - test/companies/_qa/aws/us-east-1
---

# Drift Detection: State changed outside of terraform

Apply a resource via normal flow, then simulate drift by modifying the state outside terraform. Run plan again and verify tofuwok detects the drift (has_changes=true when no code changed).

## Phases

### Phase 1: Setup — Apply a Known Resource
Create PR with a null_resource, plan, apply, merge. This establishes known state.
Captures: SETUP_PR, SETUP_SHA

### Phase 2: Simulate Drift
After merge, the resource is in state. Now taint it or modify state to simulate drift:
```bash
cd test/companies/_qa/aws/us-east-1
terraform taint null_resource.qa_{RUN_ID}
```
Or if taint isn't available, use `terraform state rm` to remove the resource from state (making terraform think it needs to recreate it).

### Phase 3: Create Drift Detection PR
Create a new PR that makes NO code changes — just an empty commit or a comment. The plan should still show changes because state drifted.
Captures: DRIFT_PR, DRIFT_SHA

### Phase 4: Wait for Plan
Poll tofuwok for plan results on DRIFT_PR.

### Phase 5: Verify Drift Detected
Assert via tofuwok API:
- Plan run status=success
- has_changes == true (drift detected — terraform wants to recreate the tainted resource)
- resources_to_add >= 1 or resources_to_change >= 1

This verifies that tofuwok surfaces drift even when no code changed.

### Phase 6: Apply + Merge
Apply to fix the drift, merge.

### Cleanup
Delete branches, release locks.
