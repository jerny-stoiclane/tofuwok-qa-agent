# Tofuwok QA

Autonomous E2E testing agent for tofuwok + tf-orchestrator-gha integration.

## Configuration

```
TOFUWOK_API=https://tofuwok.armhr.dev
TOFUWOK_TOKEN=twk_86fe627e207a6b5d2d57702d290b85583d0ea0f4e994d9d350ba82980c206112
TARGET_REPO=jerny-stoiclane/terraform-orchestrator-gha
TARGET_REPO_PATH=/Users/jernz/stoiclane/tf-orchestrator-gha
OWNER=jerny-stoiclane
REPO=terraform-orchestrator-gha
QA_BRANCH_PREFIX=test-qa/
```

## Usage

Invoke the QA agent:
```
@qa run t1-smoke/api-health
@qa run t2-single-pr/single-dir-plan
@qa run t2-single-pr/plan-apply-merge
@qa run t3-multi-pr/lock-conflict
@qa run all-t1
@qa run all-t2
@qa cleanup
```

## Conventions

- Test branches: `test-qa/{run-id}`
- PR titles: `[qa] {scenario} — {run-id}`
- Results: `results/{scenario}-{timestamp}.md`
- Agent never closes PRs — only merges (to test apply) or leaves for cleanup
- Cleanup always runs even on failure
