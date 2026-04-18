---
name: oidc-role-assumption
tier: t3
timeout: 180
dirs:
  - test/companies/_qa/aws/us-east-1
---

# OIDC Role Assumption: Verify credentials work end-to-end

Prove that the tofuwok-runner's OIDC auth actually results in valid AWS credentials that terraform can use. Uses `aws_caller_identity` to verify the assumed role identity matches what's configured.

## Phases

### Phase 1: Create Branch + PR
Add a data source that returns the caller identity + an output:
```hcl
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "arn" {
  value = data.aws_caller_identity.current.arn
}
```

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Timeout: 180s

### Phase 3: Verify Auth Succeeded
Assert via tofuwok API:
- Run status=success (not failure — auth worked)
- Plan output contains the caller identity ARN
- The ARN matches the expected OIDC role (contains the role name configured in ci.env or repo config)
- Account ID matches expected (814369619170 or whatever the test account is)

This proves: GHA OIDC token → tofuwok-runner exchange → STS AssumeRoleWithWebIdentity → terraform gets valid creds → data source returns correct identity.

### Phase 4: Verify No Drift From Null Resources
The plan should show the caller_identity data source as a read, not a change.
Assert: plan output does NOT show resources_to_add for the data source (data sources are read-only).

### Cleanup
Close PR (no apply needed — just verifying auth), delete branch, release locks.
