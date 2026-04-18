---
name: provider-auth-failure
tier: t3
timeout: 180
dirs:
  - test/companies/_qa/aws/us-east-1
---

# Provider Auth Failure: Bad OIDC role

Test dir with a valid AWS provider but auth config pointing to a non-existent role. tofuwok-runner's OIDC auth fails on AssumeRoleWithWebIdentity. Verifies tofuwok surfaces the error correctly.

## Phases

### Phase 1: Create Branch + PR
Add AWS provider + a data source that requires auth:
```hcl
data "aws_caller_identity" "current" {}
```
Configure the dir's auth to use a bad role (e.g., `arn:aws:iam::000000000000:role/does-not-exist`).

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
tofuwok-runner attempts OIDC → AssumeRole → fails.
Timeout: 180s

### Phase 3: Verify Auth Failure
Assert via tofuwok API:
- Run status=failure
- Error output contains auth-related error (AssumeRole, AccessDenied, credentials)
- has_changes=false

Assert tofuwok surfaced it:
- Failure status/check run posted
- PR comment shows the auth error clearly (not a raw stack trace)

### Cleanup
Close PR, delete branch, release any locks.
