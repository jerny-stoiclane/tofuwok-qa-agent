---
name: no-changes
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# No Changes: Plan shows no diff

Create a PR that touches a file but doesn't change terraform state (e.g., add a comment). Plan should succeed with has_changes=false. Lock behavior on no-changes is a known edge case.

## Phases

### Phase 1: Create Branch + PR
Add only a comment to variables.tf (NOT a null_resource).
```bash
echo '# no-change-test' >> test/companies/bravo/snowflake/variables.tf
```
Create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan via Tofuwok API
Poll until plan run reaches terminal status.
Timeout: 180s

### Phase 3: Verify No-Changes Plan
Assert via tofuwok API:
- Run exists with status=success, has_changes=false
- resources_to_add=0, resources_to_change=0, resources_to_destroy=0
- Lock behavior: check if lock is acquired or released (no-changes may auto-release)
- Commit status posted

### Phase 4: Merge PR
No apply needed — no changes to apply.
```bash
gh pr merge PR_NUMBER --repo TARGET_REPO --merge
```

### Phase 5: Verify Lock Released
Assert: no lock remains after merge

### Cleanup
Delete branch if not auto-deleted.
