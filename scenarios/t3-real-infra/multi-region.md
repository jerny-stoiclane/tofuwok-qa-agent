---
name: multi-region
tier: t3
timeout: 600
dirs:
  - test/companies/_qa/aws/us-east-1
  - test/companies/_qa/aws/us-west-2
---

# Multi-Region: Same account, different regions

PR touches dirs in two regions. Verify the runner resolves the correct region per dir and both plans/applies succeed independently.

## Phases

### Phase 1: Create Branch + PR
Add an SSM parameter to both region dirs:
```hcl
# us-east-1/main.tf
resource "aws_ssm_parameter" "region_test" {
  name  = "/qa/${RUN_ID}/us-east-1"
  type  = "String"
  value = "us-east-1"
}

# us-west-2/main.tf
resource "aws_ssm_parameter" "region_test" {
  name  = "/qa/${RUN_ID}/us-west-2"
  type  = "String"
  value = "us-west-2"
}
```

Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plans
Both dirs should get separate plan runs.
Assert:
- 2 plan runs dispatched (one per dir)
- Both succeed with has_changes=true
- Each plan's provider uses the correct region

### Phase 3: Apply Both
Trigger apply for both dirs.
Assert:
- Both applies succeed
- SSM parameters created in their respective regions

### Phase 4: Merge + Cleanup
Merge PR. Create destroy PR to clean up both SSM params.

### Prerequisites
- `test/companies/_qa/aws/us-west-2/` dir exists with provider config for us-west-2
- Same OIDC role works in both regions (role is global, resources are regional)
