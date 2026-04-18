---
name: multi-account
tier: t3
timeout: 600
dirs:
  - test/companies/_qa/aws/us-east-1
  - test/companies/_qa-secondary/aws/us-east-1
---

# Multi-Account: Different companies, different AWS accounts

PR touches dirs in two different companies with different AWS accounts. Verify the runner resolves a different OIDC role per dir based on each company's auth config.

## Phases

### Phase 1: Create Branch + PR
Add an SSM parameter to both account dirs:
```hcl
# _qa/aws/us-east-1/main.tf (account A)
resource "aws_ssm_parameter" "account_test" {
  name  = "/qa/${RUN_ID}/account-a"
  type  = "String"
  value = "primary-account"
}

# _qa-secondary/aws/us-east-1/main.tf (account B)
resource "aws_ssm_parameter" "account_test" {
  name  = "/qa/${RUN_ID}/account-b"
  type  = "String"
  value = "secondary-account"
}
```

Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plans
Assert:
- 2 plan runs dispatched (one per dir)
- Each plan used a DIFFERENT assumed role (check caller identity in output)
- Both succeed with has_changes=true

### Phase 3: Verify Role Isolation
Assert:
- _qa dir's plan assumed role from _qa/ci.env (account A)
- _qa-secondary dir's plan assumed role from _qa-secondary/ci.env (account B)
- No cross-account credential leakage

### Phase 4: Apply Both + Merge
Trigger apply for both. Verify resources created in correct accounts.
Merge. Destroy PR to clean up.

### Prerequisites
- `test/companies/_qa/ci.env` with account A role
- `test/companies/_qa-secondary/ci.env` with account B role
- Both OIDC roles configured with GitHub trust for this repo
- Both roles have SSM permissions in their respective accounts
