---
name: api-health
tier: t1
timeout: 30
---

# Smoke: API Health

## Phases

### Phase 1: Tofuwok API
Check tofuwok is up and responding.
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/health"
```
Assert: response is "ok"

### Phase 2: Repo Registered
Check target repo is registered with gha execution mode.
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/repos" | \
  jq '.[] | select(.repo=="terraform-orchestrator-gha")'
```
Assert: repo found, settings.execution_mode == "gha"

### Phase 3: GitHub Access
Check gh CLI can access the target repo.
```bash
gh repo view jerny-stoiclane/terraform-orchestrator-gha --json name
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/actions/workflows --jq '.workflows[].name'
```
Assert: repo accessible, workflows listed (should include "Tofuwok", "Tofuwok Plan", "Tofuwok Apply")

### Phase 4: Clean State
Check no stale locks or test PRs.
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/locks/jerny-stoiclane/terraform-orchestrator-gha" | jq length
gh pr list --repo jerny-stoiclane/terraform-orchestrator-gha --search "test-qa/" --state open --json number | jq length
```
Assert: 0 locks, 0 test PRs (or cleanup if found)
