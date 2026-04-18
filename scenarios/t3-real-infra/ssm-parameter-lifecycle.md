---
name: ssm-parameter-lifecycle
tier: t3
timeout: 600
dirs:
  - test/companies/_qa/aws/us-east-1
---

# SSM Parameter Lifecycle: Real AWS resource create + destroy

Full lifecycle with a real AWS resource: plan shows +1, apply creates it, verify it exists in AWS, destroy plan shows -1, apply destroys it.

Uses SSM Parameter Store — always free, sub-second create/destroy.

## Phases

### Phase 1: Create Branch + PR (Add Resource)
```hcl
resource "aws_ssm_parameter" "qa_test" {
  name  = "/qa/${RUN_ID}/test-param"
  type  = "String"
  value = "hello from qa agent"
  tags = {
    ManagedBy = "tofuwok-qa"
    RunID     = "${RUN_ID}"
  }
}
```

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Assert via tofuwok: status=success, has_changes=true, resources_to_add=1

### Phase 3: Apply
Trigger apply, wait for completion.
Assert: status=success

### Phase 4: Verify Resource Exists in AWS
The QA agent can verify via tofuwok runs output, or if the test dir has an output:
```hcl
output "param_name" { value = aws_ssm_parameter.qa_test.name }
```
Check that the apply output confirms creation.

### Phase 5: Merge
Merge PR. Lock released.

### Phase 6: Create Destroy PR
New PR that removes the resource. Plan should show resources_to_destroy=1.

### Phase 7: Apply Destroy + Merge
Apply the destroy, verify resource removed, merge.

### Cleanup
Delete branches. Verify SSM parameter no longer exists (tofuwok apply output should confirm destruction).
