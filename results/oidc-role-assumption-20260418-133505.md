# oidc-role-assumption — 20260418-133505

- Started: 2026-04-18T13:35:05Z
- Target: jerny-stoiclane/terraform-orchestrator-gha
- Dir: test/companies/_qa/aws/us-east-1
- Expected Role: github-actions-tf-orchestrator-gha
- Expected Account: 814369619170

- PR: #33
- HEAD SHA: 548de0de8ec70380540a18f441cd5ad15372ffca

## Phase 1: Create Branch + PR

- Status: PASS
- Steps:
  - [x] Created branch test-qa/20260418-133505
  - [x] Added aws_caller_identity data source + null_resource to main.tf
  - [x] Created PR #33
    - Details: https://github.com/jerny-stoiclane/terraform-orchestrator-gha/pull/33

## Phase 2: Wait for Plan

- Status: PASS
- Duration: 136s
- Steps:
  - [x] Plan completed successfully
    - Details: polled `bin/twk wait-plans` 10 times over 136s
    - Run ID: 2aa81339-79f3-4746-848a-9be26d186a9a
    - Result: success, has_changes=true, +1 ~0 -0
    - Terraform version: 1.14.8

## Phase 3: Verify Auth Succeeded

- Status: PASS
- Steps:
  - [x] Run status = success (auth worked)
    - Details: `bin/twk runs --pr 33 --type plan` → status=success
  - [x] Plan output contains caller identity ARN
    - Details: Output contains `arn:aws:sts::814369619170:assumed-role/github-actions-tf-orchestrator-gha/tofuwok-_qa-aws-us-east-1-1776519479`
  - [x] ARN matches expected OIDC role (github-actions-tf-orchestrator-gha)
    - Details: ARN contains `github-actions-tf-orchestrator-gha` which matches ci.env IAM_ROLE
  - [x] Account ID matches expected (814369619170)
    - Details: Output contains `account_id = "814369619170"`

## Phase 4: Verify No Drift From Null Resources

- Status: PASS
- Steps:
  - [x] Data source is read-only, not a resource change
    - Details: Plan shows `resources_to_add=1` (the null_resource only). The `aws_caller_identity` data source appears only in output changes, not as a resource action. Plan text confirms only `null_resource.qa_20260418_133505 will be created`.

## Cleanup

- Closed PR #33
- Deleted branch test-qa/20260418-133505
- No remaining locks

## Summary

- Result: PASS
- Duration: ~160s
- Passed: 5/5
- Failed: 0/5

### Key Evidence
- OIDC auth chain proven: GHA OIDC token -> tofuwok-runner -> STS AssumeRoleWithWebIdentity -> terraform got valid creds
- Assumed role ARN: `arn:aws:sts::814369619170:assumed-role/github-actions-tf-orchestrator-gha/tofuwok-_qa-aws-us-east-1-1776519479`
- Account ID: `814369619170` (matches ci.env)
- Role name: `github-actions-tf-orchestrator-gha` (matches ci.env IAM_ROLE)
