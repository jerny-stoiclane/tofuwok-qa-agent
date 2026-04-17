---
name: provider-auth-failure
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Provider Auth Failure: Valid provider, bad credentials

Create PR with a resource that requires AWS credentials but the plan runs without valid creds (or with intentionally bad creds). Terraform init succeeds (provider downloads fine) but plan fails on authentication.

## Phases

### Phase 1: Create Branch + PR
Add an AWS resource that requires real credentials to plan:
```hcl
data "aws_caller_identity" "bad_creds" {}
```
This requires the AWS provider and valid credentials to plan. Since the test runner uses a role that may not have this permission, or the resource references a non-existent account, plan should fail with an auth error.

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll tofuwok for plan run. Should reach terminal status (failure).
Timeout: 180s

### Phase 3: Verify Auth Failure
Assert via tofuwok API:
- Run exists with status=failure
- Error output mentions authentication, credentials, or access denied
- has_changes=false (plan never completed)
- Lock behavior: check if lock was acquired then released, or never acquired

### Phase 4: Verify Error Surfaced
Assert:
- Tofuwok posted failure status (commit status or check run)
- PR comment contains error information about the auth failure

### Cleanup
Close PR (don't merge — bad config), delete branch, release any locks.
