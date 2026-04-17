---
name: clean-state
tier: t1
timeout: 30
---

# Smoke: Clean State

## Phases

### Phase 1: No Stale Locks
Verify no orphan locks exist.
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/locks/jerny-stoiclane/terraform-orchestrator-gha"
```
Assert: empty array or only locks from active open PRs

### Phase 2: No Stale Test PRs
Verify no leftover test PRs from previous runs.
```bash
gh pr list --repo jerny-stoiclane/terraform-orchestrator-gha --search "test-qa/" --state open
```
Assert: 0 open test PRs

### Phase 3: No Stuck Runs
Verify no runs stuck in "running" state for >10 minutes.
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/runs/jerny-stoiclane/terraform-orchestrator-gha"
```
Assert: no runs with status="running" older than 10 minutes
