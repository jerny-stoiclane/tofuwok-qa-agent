# provider-auth-failure — 20260418-031608

- Started: 2026-04-18T03:16:08Z
- Target: jerny-stoiclane/terraform-orchestrator-gha
- PR: #30 (v3 — previous attempts #28 closed without PR record, #29 closed prematurely)
- HEAD SHA: 70797a2b0d759e2778cf7514c114de871488d865
- Branch: test-qa/20260418-031608-v3 (STILL OPEN)

## Phase 1: Create Branch + PR

- Status: PASS
- Steps:
  - [x] Created branch with bad AWS provider role in bravo/snowflake
  - [x] Created PR #30

## Phase 2: Wait for Plan

- Status: PASS (plan completed)
- Duration: ~136s
- Steps:
  - [x] Plan completed: status=failure, has_changes=false

## Phase 3: Verify Auth Failure

- Status: FAIL
- Steps:
  - [ ] Error should be provider auth related (AssumeRole, AccessDenied)
    - **ACTUAL:** S3 backend init failure — "No valid credential sources found"
    - The GHA runner has no OIDC creds for the S3 backend, so `terraform init` fails before
      the AWS provider is ever configured or the plan phase begins.

      ```text
      │ Error: No valid credential sources found
      │ Please see https://developer.hashicorp.com/terraform/language/backend/s3
      │ Error: failed to refresh cached credentials, no EC2 IMDS role found,
      │ operation error ec2imds: GetMetadata, failed to get API token
      ```

  - [x] Run status=failure in tofuwok API
  - [x] Commit status tofuwok/plan = failure
  - [x] PR comment posted ("Terraform Plan — Failed")
  - [ ] PR comment should contain auth-specific error detail
    - **ACTUAL:** generic "Failed" table, no error text in comment

## Artifacts (LEFT IN PLACE)

- PR: #30 (OPEN) — https://github.com/jerny-stoiclane/terraform-orchestrator-gha/pull/30
- Branch: test-qa/20260418-031608-v3
- Lock: none (plan failed)

## Issues Found

1. **Test design issue:** bravo/snowflake uses S3 backend which needs AWS creds for init.
   Provider auth failure can't be tested here — init fails first.
   Scenario file updated to use `test/companies/_qa/aws/us-east-1` instead.

2. **PR #28 not registered in tofuwok PRs API** — transient, PR #29 and #30 registered fine.
   Likely caused by 502 during that period.

3. **PR comment lacks error details** — just shows "Failed" with no error text.
   Developer has to click into tofuwok UI or check run logs to see what went wrong.

## Summary

- Result: FAIL
- Duration: ~140s
- Passed: 4/6
- Failed: 2/6
