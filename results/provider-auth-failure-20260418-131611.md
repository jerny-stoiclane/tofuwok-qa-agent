# provider-auth-failure — 20260418-131611

- Started: 2026-04-18T13:16:11Z
- Target: jerny-stoiclane/terraform-orchestrator-gha
- PR: #31
- HEAD SHA: 9ec8df99dc8ffe04291bac35d8ac747339b69cec
- Dir: test/companies/_qa/aws/us-east-1

## Phase 1: Create Branch + PR

- Status: PASS
- Steps:
  - [x] Branch test-qa/20260418-131611 pushed (commit 9ec8df9)
  - [x] Created PR #31 — https://github.com/jerny-stoiclane/terraform-orchestrator-gha/pull/31
  - [x] Dir test/companies/_qa/aws/us-east-1 contains: backend.tf, ci.s3.tfbackend, main.tf (bad role ARN)

## Phase 2: Wait for Plan

- Status: PASS
- Duration: ~135s (detected at poll 7, completed at poll ~14)
- Steps:
  - [x] Tofuwok created run 79331005-6387-4a3a-8cd5-44fe4d559bb3
    - Details: run_type=plan, dir=test/companies/_qa/aws/us-east-1
  - [x] Plan completed with status=failure
    - Details: started 08:19:34, completed 08:20:49 (~75s execution)

## Phase 3: Verify Auth Failure

- Status: PASS
- Steps:
  - [x] Run status=failure
    - Details: `GET /api/v1/runs/79331005-...` → status=failure
  - [x] Error output contains auth-related error
    - Details: Output contains: "STS AssumeRoleWithWebIdentity failed", "StatusCode: 403", "AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity"

      ```text
      Error resolving AWS credentials: STS AssumeRoleWithWebIdentity failed: operation error STS:
      AssumeRoleWithWebIdentity, https response error StatusCode: 403, RequestID: a801cdb1-...,
      api error AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity — check the
      trust policy on role arn:aws:iam::814369619170:role/github-actions/github-actions-tf-orchestrator-spike
      to ensure it trusts token.actions.githubusercontent.com
      ```

  - [x] has_changes=false (no plan output since init/auth failed)
    - Details: has_changes=false, resources_to_add=0, resources_to_change=0, resources_to_destroy=0
  - [x] Failure commit status posted
    - Details: `gh api commits/{sha}/statuses` → tofuwok/plan state=failure, description="1 failed, 0 succeeded"
  - [x] PR comment shows auth error clearly (not raw stack trace)
    - Details: PR comment posted with ":x: Terraform Plan — Failed" summary table, links to run in tofuwok UI
  - [x] No lock acquired (plan failed, no lock needed)
    - Details: `bin/twk locks --dir test/companies/_qa/aws/us-east-1` → []

## Timeline

- 00:00 Branch test-qa/20260418-131611 already pushed
- 00:02 Created PR #31
- 01:00 Tofuwok detected PR, created plan run (pending → running)
- 02:15 Plan completed with status=failure (auth error)
- 02:20 Verified all assertions

## Summary

- Result: PASS
- Duration: ~140s
- Passed: 6/6
- Note: This test verifies tofuwok correctly surfaces AWS OIDC auth failures. The error message
  is clear and actionable — it tells the user exactly which role failed and what to check (trust policy).
  The error was "AssumeRoleWithWebIdentity 403 AccessDenied" as expected when using a role
  that doesn't trust the GHA OIDC provider.
