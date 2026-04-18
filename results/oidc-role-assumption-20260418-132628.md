# t3-real-infra/oidc-role-assumption — 20260418-132628

- Started: 2026-04-18T13:26:28Z
- Target: jerny-stoiclane/terraform-orchestrator-gha
- PR: (pending)
- HEAD SHA: (pending)


## Phase 1: Create Branch + PR

- Status: PASS
- Steps:
  - [x] Created _qa test infrastructure on main (commit 3f3b170)
  - [x] Created branch test-qa/20260418-132628
  - [x] Added aws_caller_identity data source + outputs to main.tf
  - [x] Created PR #32
    - Details: https://github.com/jerny-stoiclane/terraform-orchestrator-gha/pull/32
  - HEAD SHA: 58196895b3732b2ab3a0b6a5339c8c73905340a1

## Phase 2: Wait for Plan

- Status: PASS (plan completed, but with failure status)
- Duration: ~90s
- Steps:
  - [x] Plan run detected: bec5fa5e-643a-4325-becf-16d1c23b9458
  - [x] Plan completed in ~67s (running -> failure)
  - Run details:
    - Dir: test/companies/_qa/aws/us-east-1
    - Status: failure
    - Tool: terraform 1.14.8
    - GHA Run: https://github.com/jerny-stoiclane/terraform-orchestrator-gha/actions/runs/24605710932

## Phase 3: Verify Auth

- Status: FAIL
- Steps:
  - [ ] FAIL: Plan status=failure (expected success)
    - Error: `STS AssumeRoleWithWebIdentity failed: AccessDenied`
    - Role used: `arn:aws:iam::814369619170:role/github-actions/github-actions-tf-orchestrator-spike` (repo-level oidc_role_arn)
    - The trust policy on this role does not trust `token.actions.githubusercontent.com` for the `terraform-orchestrator-gha` repo
  - [ ] FAIL: Cannot verify caller identity ARN (plan never reached terraform plan stage)
  - [ ] FAIL: Cannot verify account ID (plan never reached terraform plan stage)

### Root Cause Analysis

The repo-level `oidc_role_arn` is set to `arn:aws:iam::814369619170:role/github-actions/github-actions-tf-orchestrator-spike`. This role's trust policy only allows the `tf-orchestrator-spike` repo to assume it, not `terraform-orchestrator-gha`.

Two issues:
1. **IAM trust policy gap**: The role needs its trust policy updated to include `repo:jerny-stoiclane/terraform-orchestrator-gha:*` as an allowed subject
2. **ci.env not used for OIDC**: The `ci.env` file has `IAM_ROLE=...github-actions-tf-orchestrator-gha` but the runner used the repo-level `oidc_role_arn` instead. This may be by design (repo-level overrides per-dir) or a bug.

## Phase 4: Verify No Drift

- Status: SKIP (plan failed before terraform plan stage)


## Cleanup

- [x] PR #32 closed with branch deleted
- [x] No orphan locks
- [x] Returned to main

## Timeline

- 00:00 Created _qa test infrastructure on main
- 00:30 Created branch test-qa/20260418-132628
- 00:45 Created PR #32
- 01:10 Plan run detected (bec5fa5e)
- 02:20 Plan completed: failure (OIDC AccessDenied)
- 02:25 Verified failure: IAM trust policy does not allow this repo
- 02:30 Closed PR #32, deleted branch

## Summary

- Result: FAIL
- Duration: ~150s
- Passed: 0/3
- Failed: 3/3 (all auth assertions failed due to OIDC trust policy misconfiguration)
- Root cause: Infrastructure, not tofuwok — the IAM role's trust policy needs updating
