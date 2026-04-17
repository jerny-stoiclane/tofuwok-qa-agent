---
name: missing-provider
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Missing Provider: Plan fails on init

Create PR with a resource that requires a provider not in the lock file. Terraform init or plan should fail.

## Phases

### Phase 1: Create Branch + PR
Add a resource using a non-existent provider:
```hcl
resource "fake_nonexistent_resource" "test" {
  name = "should-fail"
}
```
Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll until plan run reaches terminal status (should be failure).
Timeout: 180s

### Phase 3: Verify Failed Plan
Assert via tofuwok API:
- Run status=failure
- Error output mentions missing provider or init failure
- has_changes=false

### Phase 4: Verify Error Surfaced
Assert:
- Commit status shows failure
- PR comment contains error information

### Cleanup
Close PR, delete branch, release any locks.
