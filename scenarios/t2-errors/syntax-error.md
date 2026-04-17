---
name: syntax-error
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Syntax Error: Plan fails with HCL error

Create PR with invalid HCL. Plan should fail and tofuwok should surface the error correctly.

## Phases

### Phase 1: Create Branch + PR
Add invalid terraform to main.tf:
```hcl
resource "null_resource" "bad" {
  this_is_not_valid = {
```
Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll until plan run reaches terminal status (should be failure).
Timeout: 180s

### Phase 3: Verify Failed Plan
Assert via tofuwok API:
- Run exists with status=failure
- Run has error output containing the syntax error message
- has_changes=false (plan never got to the change detection phase)
- Lock should NOT be acquired (plan failed before lock)
  OR lock acquired then released (depends on tofuwok behavior)

### Phase 4: Verify Error Surfaced
Assert:
- Commit status shows failure
- PR comment contains error information

### Cleanup
Close PR (don't merge — bad HCL), delete branch, release any locks.
