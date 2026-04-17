---
name: workflows-current
tier: t1
timeout: 30
orchestrator: gha
---

# Smoke: Workflows Current on Main

Verify the tofuwok runner workflows on the target repo's main branch are present and match the expected structure.

## Phases

### Phase 1: Affected Workflow Exists
```bash
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/contents/.github/workflows/affected.yaml --jq '.name'
```
Assert: file exists on main

### Phase 2: Plan Workflow Exists with workflow_dispatch
```bash
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/contents/.github/workflows/tofuwok-plan.yaml --jq '.content' | base64 -d | head -20
```
Assert: file exists, has `workflow_dispatch` trigger, has `run-id`, `dir`, `sha`, `pr-number` inputs

### Phase 3: Apply Workflow Exists with workflow_dispatch
```bash
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/contents/.github/workflows/tofuwok-apply.yaml --jq '.content' | base64 -d | head -20
```
Assert: file exists, has `workflow_dispatch` trigger, has `run-id`, `dir`, `sha`, `pr-number` inputs

### Phase 4: Workflows Reference tofuwok Actions
```bash
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/contents/.github/workflows/affected.yaml --jq '.content' | base64 -d | grep "jerny-stoiclane/tofuwok/actions/"
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/contents/.github/workflows/tofuwok-plan.yaml --jq '.content' | base64 -d | grep "jerny-stoiclane/tofuwok/actions/"
```
Assert: workflows reference `jerny-stoiclane/tofuwok/actions/setup`, `actions/affected`, `actions/plan`, `actions/apply`

### Phase 5: TOFUWOK_API_URL Variable Set
```bash
gh variable list --repo jerny-stoiclane/terraform-orchestrator-gha --json name,value --jq '.[] | select(.name=="TOFUWOK_API_URL")'
```
Assert: TOFUWOK_API_URL is set and points to a reachable endpoint
