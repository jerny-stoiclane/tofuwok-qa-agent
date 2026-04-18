---
name: wrong-region
tier: t3
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Wrong Region: Provider region mismatch

Test dir configured for a region the OIDC role doesn't have access to, or a region that doesn't support the requested service. Plan should fail with a region-related error.

## Phases

### Phase 1: Create Branch + PR
Add AWS provider with a nonsense or restricted region + a resource:
```hcl
provider "aws" {
  region = "af-south-1"  # Cape Town — may not be enabled on the account
}

resource "aws_ssm_parameter" "test" {
  name  = "/qa/wrong-region-test"
  type  = "String"
  value = "should-fail"
}
```

Push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Timeout: 180s

### Phase 3: Verify Region Error
Assert via tofuwok API:
- Run status=failure
- Error mentions region, opt-in, or endpoint not available

Assert tofuwok surfaced it:
- Failure posted to PR

### Cleanup
Close PR, delete branch, release any locks.
