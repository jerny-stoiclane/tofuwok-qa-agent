# workflows-current — 20260417-140957

- Started: 2026-04-17T14:10:10Z
- Target: jerny-stoiclane/terraform-orchestrator-gha

## Phase 1: Affected Workflow Exists

- Status: PASS
- Steps:
  - [x] affected.yaml exists on main
    - Details: `gh api .../contents/.github/workflows/affected.yaml` → name=affected.yaml

## Phase 2: Plan Workflow

- Status: PASS
- Steps:
  - [x] tofuwok-plan.yaml exists with workflow_dispatch
    - Details: has `workflow_dispatch` trigger with inputs: run-id, dir (sha, pr-number in remaining lines)

## Phase 3: Apply Workflow

- Status: PASS
- Steps:
  - [x] tofuwok-apply.yaml exists with workflow_dispatch
    - Details: has `workflow_dispatch` trigger with inputs: run-id, dir

## Phase 4: Workflows Reference tofuwok Actions

- Status: PASS
- Steps:
  - [x] affected.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `actions/affected@main`
  - [x] tofuwok-plan.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `actions/plan@main`

## Phase 5: TOFUWOK_API_URL Variable

- Status: PASS
- Steps:
  - [x] TOFUWOK_API_URL set and correct
    - Details: `gh variable list` → name=TOFUWOK_API_URL, value=https://tofuwok.armhr.dev

## Summary

- Result: PASS
- Duration: 3s
- Passed: 6/6
