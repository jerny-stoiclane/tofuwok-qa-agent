---
name: gh-access
tier: t1
timeout: 30
---

# Smoke: GitHub Access

## Phases

### Phase 1: Authentication
Verify gh CLI is authenticated.
```bash
gh auth status
```
Assert: authenticated, shows active account

### Phase 2: Repo Access
Verify read/write access to target repo.
```bash
gh repo view jerny-stoiclane/terraform-orchestrator-gha --json name,defaultBranchRef
```
Assert: repo accessible, default branch is main

### Phase 3: Workflows Present
Verify tofuwok workflows exist on main.
```bash
gh api repos/jerny-stoiclane/terraform-orchestrator-gha/actions/workflows --jq '.workflows[].name'
```
Assert: "Tofuwok", "Tofuwok Plan", "Tofuwok Apply" all present

### Phase 4: Can Create Branches
Verify push access (create and immediately delete a test branch).
```bash
cd /Users/jernz/stoiclane/tf-orchestrator-gha
git checkout main && git pull
git checkout -b test-qa/smoke-probe
git push origin test-qa/smoke-probe
git push origin --delete test-qa/smoke-probe
git checkout main
```
Assert: branch created and deleted without error
