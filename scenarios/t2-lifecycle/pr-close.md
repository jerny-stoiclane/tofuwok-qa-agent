---
name: pr-close
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# PR Close: Locks released on close without merge

Create PR, plan acquires lock, close PR without merging. Lock should be released.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER

### Phase 2: Wait for Plan
Poll until plan completes. Verify lock acquired.

### Phase 3: Close PR (Without Merge)
```bash
gh pr close PR_NUMBER --repo TARGET_REPO
```

### Phase 4: Wait for Lock Release
Poll tofuwok locks API. Lock should be released within 30s of PR close.
Timeout: 60s

### Phase 5: Verify
Assert:
- Lock for test/companies/bravo/snowflake is released (no lock found)
- No apply was dispatched (no apply runs exist for this PR)

### Cleanup
Delete branch.
