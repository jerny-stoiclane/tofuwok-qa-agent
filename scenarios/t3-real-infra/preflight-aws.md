---
name: preflight-aws
tier: t3
timeout: 60
---

# T3 Preflight: Verify AWS setup before running real infra tests

Must pass before any other T3 scenario. Verifies the test dir exists, auth is configured, and the runner can assume the role.

## Phases

### Phase 1: Test Dir Exists
Verify `test/companies/_qa/aws/us-east-1/` exists in the repo on main with the required files.

Assert:
- Directory exists
- Has main.tf
- Has variables.tf or backend.tf

### Phase 2: Auth Config Exists
Verify either:
- `test/companies/_qa/ci.env` has a valid `IAM_ROLE`
- OR tofuwok repo config has `default_aws_role` set

Check via tofuwok API:
```bash
curl ... /api/v1/repos/jerny-stoiclane/terraform-orchestrator-gha/config/effective
```

Assert: a role ARN is resolvable for the `_qa/aws/us-east-1` dir

### Phase 3: Create a Smoke PR
Create a minimal PR that adds only:
```hcl
data "aws_caller_identity" "preflight" {}
```
This tests the full chain: PR → tofuwok → dispatch plan → runner OIDC auth → terraform plan.

### Phase 4: Verify Auth Works
Assert via tofuwok API:
- Plan run exists with status=success (auth worked, data source resolved)
- OR status=failure with a CLEAR error about what's wrong (role not found, OIDC failed, etc.)

If failure: report exactly what failed and stop. Don't run other T3 scenarios.

### Cleanup
Close PR, delete branch.
