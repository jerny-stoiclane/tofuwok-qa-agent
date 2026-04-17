# workflows-current — 20260417-123301

- Started: 2026-04-17T12:33:01Z
- Target: jerny-stoiclane/terraform-orchestrator-gha

## Phase 1: Affected Workflow Exists

- Status: PASS
- Steps:
  - [x] affected.yaml exists on main
    - Details: `gh api .../contents/.github/workflows/affected.yaml` → name: affected.yaml

## Phase 2: Plan Workflow — workflow_dispatch

- Status: PASS
- Steps:
  - [x] tofuwok-plan.yaml has workflow_dispatch with required inputs
    - Details: Has `workflow_dispatch` trigger with inputs: run-id, dir (confirmed via base64 decode of file content)

## Phase 3: Apply Workflow — workflow_dispatch

- Status: PASS
- Steps:
  - [x] tofuwok-apply.yaml has workflow_dispatch with required inputs
    - Details: Has `workflow_dispatch` trigger with inputs: run-id, dir (confirmed via base64 decode of file content)

## Phase 4: Workflows Reference tofuwok Actions

- Status: PASS
- Steps:
  - [x] affected.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `jerny-stoiclane/tofuwok/actions/affected@main`
  - [x] tofuwok-plan.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `jerny-stoiclane/tofuwok/actions/plan@main`

## Phase 5: TOFUWOK_API_URL Variable Set

- Status: PASS
- Steps:
  - [x] Variable exists and points to correct endpoint
    - Details: `gh variable list` → TOFUWOK_API_URL = https://tofuwok.armhr.dev

## Summary

- Result: PASS
- Passed: 5/5
