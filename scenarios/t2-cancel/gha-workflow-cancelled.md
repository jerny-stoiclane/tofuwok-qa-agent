---
name: gha-workflow-cancelled
tier: t2
timeout: 300
orchestrator: gha
dirs:
  - test/companies/bravo/snowflake
---

# GHA Workflow Cancelled: Runner dies mid-execution

Simulate what happens when the GHA workflow is cancelled externally (someone clicks Cancel in GitHub Actions UI, or the runner crashes). Tofuwok should detect the orphaned run via heartbeat timeout.

## Phases

### Phase 1: Create Branch + PR
Add null_resource, push, create PR, wait for plan to start running.
Captures: PR_NUMBER, HEAD_SHA, RUN_ID

### Phase 2: Cancel the GHA Workflow
Cancel the running GHA workflow via GitHub API:
```bash
gh run cancel WORKFLOW_RUN_ID --repo TARGET_REPO
```
This kills the runner — tofuwok doesn't get a completion callback.

### Phase 3: Wait for Heartbeat Timeout
Tofuwok should detect the orphaned run when heartbeat stops (heartbeat expiry is ~90s).
Wait up to 120s.

### Phase 4: Verify Orphan Detection
Assert via tofuwok API:
- Run status eventually transitions from "running" to "failure" or "cancelled" (heartbeat timeout)
- Run status_reason mentions heartbeat or timeout
- Lock released after orphan detection

### Phase 5: Verify Recovery
Push empty commit to retrigger. Verify tofuwok dispatches a new plan and it runs cleanly.

### Cleanup
Close or merge PR. Delete branch, release locks.
