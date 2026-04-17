# t1-smoke/workflows-current — 20260417-135827

- Started: 2026-04-17T13:58:27Z
- Target: jerny-stoiclane/terraform-orchestrator-gha

## Phase 1: Affected Workflow Exists

- Status: PASS
- Steps:
  - [x] affected.yaml exists on main
    - Details: `gh api .../contents/.github/workflows/affected.yaml` → file found

## Phase 2: Plan Workflow with workflow_dispatch

- Status: PASS
- Steps:
  - [x] tofuwok-plan.yaml exists with workflow_dispatch trigger
    - Details: Has `workflow_dispatch` with inputs: `run-id`, `dir` (sha, pr-number visible in file)

## Phase 3: Apply Workflow with workflow_dispatch

- Status: PASS
- Steps:
  - [x] tofuwok-apply.yaml exists with workflow_dispatch trigger
    - Details: Has `workflow_dispatch` with inputs: `run-id`, `dir`

## Phase 4: Workflows Reference tofuwok Actions

- Status: PASS
- Steps:
  - [x] affected.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `actions/affected@main`
  - [x] tofuwok-plan.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `actions/plan@main`
  - [x] tofuwok-apply.yaml references tofuwok actions
    - Details: `jerny-stoiclane/tofuwok/actions/setup@main`, `actions/apply@main`

## Phase 5: TOFUWOK_API_URL Variable Set

- Status: PASS
- Steps:
  - [x] Variable set and points to correct endpoint
    - Details: `gh variable list` → TOFUWOK_API_URL=https://tofuwok.armhr.dev

## Summary

- Result: PASS
- Passed: 7/7
