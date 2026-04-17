---
name: pr-close-during-apply
tier: t2
timeout: 300
dirs:
  - test/companies/bravo/snowflake
---

# PR Close During Apply: Race condition

Trigger apply, then close the PR while apply may still be running. Tests what happens to in-flight applies and locks when a PR is closed.

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

Assert — State safety:
- If apply succeeded but PR was closed (not merged): state changed but code wasn't merged
- Tofuwok SHOULD raise a GitHub issue or warning: "Apply succeeded for {dir} on PR #{N} but PR was closed without merge. Infrastructure state has diverged from code on main."
- Check if tofuwok created an issue on the repo with this warning
- The next plan on this dir (from any PR) should detect the state change as drift

### Cleanup
Delete branch. Release any remaining locks.
Note: if apply succeeded and changed state, the next test may see drift. That's expected.
