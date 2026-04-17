---
name: rapid-commits
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# Rapid Commits: Supersession

Push 3 commits in quick succession to the same PR. Only the latest commit's plan should be the final result. Earlier plans should be superseded.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER

### Phase 2: Rapid Push — 3 Commits
Push 3 commits ~5 seconds apart:
```bash
for i in 1 2 3; do
  echo "resource \"null_resource\" \"rapid_${i}\" { triggers = { v = \"${i}\" } }" >> test/companies/bravo/snowflake/main.tf
  git add -A && git commit -m "test: rapid commit $i" && git push
  sleep 5
done
```
Captures: FINAL_SHA (from last push)

### Phase 3: Wait for Plans to Settle
Wait 120s, then check tofuwok runs.
The key assertion: the LATEST commit's plan is the one that matters.

### Phase 4: Verify Supersession
Assert via tofuwok API:
- At least one plan run with status=success for FINAL_SHA
- Earlier runs may be superseded (status=superseded) or may have completed
- Only the latest plan's lock should be active

### Phase 5: Apply + Merge
Trigger apply for the latest plan, merge.

### Cleanup
Delete branch, release locks.
