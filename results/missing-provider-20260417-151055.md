# missing-provider — 20260417-151055

- Started: 2026-04-17T15:10:55Z
- Target: jerny-stoiclane/terraform-orchestrator-gha
- PR: #17
- HEAD SHA: ad1d150e11cc566397504f7bc0d7db831f018c02

## Phase 1: Create Branch + PR

- Status: PASS
- Steps:
  - [x] Created branch test-qa/20260417-151055 with fake_nonexistent_resource in bravo/snowflake
  - [x] Created PR #17
  - [x] HEAD SHA: ad1d150e11cc566397504f7bc0d7db831f018c02

## Phase 2: Wait for Plan

- Status: PASS
- Duration: ~152s
- Steps:
  - [x] Plan completed with status=failure (expected)

## Phase 3: Verify Failed Plan

- Status: PASS
- Steps:
  - [x] Run exists with status=failure, run_type=plan
    - Details: `bin/twk runs --pr 17 --type plan` → id=97112dab, status=failure
  - [x] has_changes=false, all resource counts=0
  - [x] No lock acquired (plan failed)
    - Details: `bin/twk locks --dir test/companies/bravo/snowflake` → [] (empty)

## Phase 4: Verify Error Surfaced

- Status: PASS
- Steps:
  - [x] Commit status shows failure
    - Details: `gh api commits/{sha}/statuses` → tofuwok/plan = failure
  - [x] PR comment contains error information
    - Details: tofuwok-local[bot] posted "Terraform Plan — Failed, 1 of 1 directories failed"

## Cleanup

- Closed PR #17 (bad provider — not merged)
- Deleted branch test-qa/20260417-151055
- No locks to release

## Summary

- Result: PASS
- Duration: ~155s
- Passed: 6/6
- Failed: 0/6
