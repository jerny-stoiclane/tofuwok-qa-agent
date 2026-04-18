---
name: backend-unreachable
tier: t3
timeout: 180
dirs:
  - test/companies/_qa/aws/us-east-1
---

# Backend Unreachable: S3 backend misconfigured

Test dir with an S3 backend config pointing to a non-existent bucket. Terraform init fails because it can't access the state backend.

## Phases

### Phase 1: Create Branch + PR
Add an S3 backend config pointing to a bucket that doesn't exist:
```hcl
terraform {
  backend "s3" {
    bucket = "this-bucket-does-not-exist-qa-test"
    key    = "qa/test.tfstate"
    region = "us-east-1"
  }
}

resource "aws_ssm_parameter" "test" {
  name  = "/qa/backend-test"
  type  = "String"
  value = "should-fail-on-init"
}
```

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
terraform init should fail (can't reach backend). Plan never runs.
Timeout: 180s

### Phase 3: Verify Init Failure
Assert via tofuwok API:
- Run status=failure
- Error mentions S3, bucket, NoSuchBucket, or backend initialization
- has_changes=false (never got to plan)

Assert tofuwok surfaced it:
- Failure posted to PR
- Error message is clear (not just "exit code 1")

### Cleanup
Close PR, delete branch, release any locks.
