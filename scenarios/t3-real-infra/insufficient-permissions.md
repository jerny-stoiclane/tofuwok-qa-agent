---
name: insufficient-permissions
tier: t3
timeout: 180
dirs:
  - test/companies/_qa/aws/us-east-1
---

# Insufficient Permissions: Role lacks required IAM permissions

Auth succeeds (valid role, correct region) but the role doesn't have permission to create the requested resource. Plan may succeed (terraform plan is mostly read-only) but apply should fail.

## Phases

### Phase 1: Create Branch + PR
Add a resource the OIDC role definitely can't create:
```hcl
resource "aws_ec2_instance" "forbidden" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name = "qa-forbidden-test"
  }
}
```
The test role should only have SSM/IAM/S3 permissions, not EC2.

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Plan may succeed (terraform plan does a read, some providers allow it) or fail (depends on whether the provider checks permissions at plan time).
Timeout: 180s

### Phase 3: Verify Behavior
Assert via tofuwok API:
- If plan failed: status=failure, error mentions "unauthorized" or "AccessDenied"
- If plan succeeded: has_changes=true (terraform thinks it can create the instance)
  - In this case, trigger apply and verify APPLY fails with permission error

Assert tofuwok surfaced it:
- Error shown in PR comment
- Correct failure status posted

### Cleanup
Close PR, delete branch, release any locks. No resources should have been created.
