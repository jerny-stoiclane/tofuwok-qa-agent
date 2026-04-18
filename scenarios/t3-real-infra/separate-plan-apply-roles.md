---
name: separate-plan-apply-roles
tier: t3
timeout: 600
dirs:
  - test/companies/_qa/aws/us-east-1
---

# Separate Plan/Apply Roles: Different IAM roles for read vs write

Verify that tofuwok-runner can use a read-only role for plan and a more privileged role for apply. This is the standard security pattern — plan shouldn't need write access, apply does.

## Phases

### Phase 1: Create Branch + PR
Add a real AWS resource (SSM parameter) that requires write permissions to apply:
```hcl
resource "aws_ssm_parameter" "role_test" {
  name  = "/qa/${RUN_ID}/role-test"
  type  = "String"
  value = "testing separate roles"
}
```

The dir's auth config should specify:
- plan_role: a read-only role (can describe/list but not create)
- apply_role: a role with write permissions

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Assert via tofuwok API:
- Run status=success (read-only role can plan)
- has_changes=true
- The assumed role in the plan output matches the plan_role ARN (not the apply_role)

### Phase 3: Trigger Apply
Apply should use the apply_role (different from plan_role).

### Phase 4: Verify Apply Used Different Role
Assert:
- Apply status=success
- The role used during apply is the apply_role (check tofuwok run output or AWS CloudTrail)
- If both roles are the same: this test is a no-op, note it as "separate roles not configured"

### Phase 5: Merge + Cleanup
Merge, then create destroy PR to clean up the SSM parameter.

### Cleanup
Delete branches, release locks, destroy the SSM parameter.

## Prerequisites
- Two IAM roles configured: one read-only, one read-write
- Dir auth config supports `plan_role` vs `apply_role` (check if tofuwok-runner supports this)
- If not supported yet: this scenario documents the requirement
