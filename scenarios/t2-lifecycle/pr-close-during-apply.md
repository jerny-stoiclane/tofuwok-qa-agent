---
name: pr-close-during-apply
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# PR Close During Apply: Orphaned Apply

Trigger apply, then close the PR while apply may still be running. If the apply completes but the PR is never merged, this creates an **orphaned apply** — infrastructure state changed but the code was never merged to main. Tofuwok should detect this and raise a warning issue.

Related: tofuwok openspec — orphaned apply detection

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll tofuwok until plan completes. Verify lock acquired.

### Phase 3: Trigger Apply
```bash
bin/twk trigger --pr PR_NUMBER --action apply
```
Do NOT wait for completion.

### Phase 4: Close PR Immediately
Close the PR right after triggering apply (within seconds):
```bash
gh pr close PR_NUMBER --repo TARGET_REPO
```

### Phase 5: Wait + Observe
Wait 120s for things to settle. Then check:

Assert — Apply completion:
- Check tofuwok runs API: did the apply run finish (success or failure)?
- A running apply should NOT be cancelled by PR close (killing terraform mid-apply corrupts state)

Assert — Lock state:
- If apply completed: lock should be released (PR is closed)
- If apply was somehow cancelled: lock should still be released

Assert — Orphaned apply detection:
- If apply succeeded but PR was closed (not merged): this is an **orphaned apply**
- Tofuwok SHOULD detect the orphaned apply and raise a GitHub issue: "Orphaned apply: {dir} was applied on PR #{N} but PR was closed without merge. Infrastructure state has diverged from code on main."
- Check tofuwok API for orphaned apply status on this run
- Check if tofuwok created a GitHub issue on the repo flagging the orphan
- The next plan on this dir (from any PR) should detect the orphaned state change as drift

### Cleanup
Delete branch. Release any remaining locks.
Note: if apply succeeded and changed state, the next test may see drift. That's expected.
