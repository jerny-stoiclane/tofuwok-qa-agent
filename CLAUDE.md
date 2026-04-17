# Tofuwok QA

Autonomous E2E testing agent for tofuwok + tf-orchestrator-gha integration.

## How to Run

### Option A: Interactive (recommended for first runs)

```bash
cd ~/stoiclane/tofuwok-qa
claude
```

Then in the session, @-mention the agent:

```
qa t1-smoke/api-health
```

You'll see each Bash command and can approve/deny via Claude Code's permission system.

### Option B: Headless via CLI

```bash
cd ~/stoiclane/tofuwok-qa
claude -p "qa t1-smoke/api-health" --agent qa
```

### Option C: Fully non-interactive (CI-friendly)

```bash
cd ~/stoiclane/tofuwok-qa
claude --bare -p "qa t2-single-pr/plan-apply-merge" --agent qa --allowedTools "Read,Write,Bash,Glob,Grep"
```

`--bare` skips environment discovery for faster, reproducible runs.

### 3. Available commands

| Command | What it does |
|---------|-------------|
| `qa t1-smoke/api-health` | Smoke test — check tofuwok API, repo registration, gh access |
| `qa t2-single-pr/single-dir-plan` | Create PR with 1 dir change, verify plan + lock |
| `qa t2-single-pr/multi-dir-plan` | Create PR with all 6 dirs, verify all plans + locks |
| `qa t2-single-pr/plan-apply-merge` | Full cycle: plan → merge → apply → verify cleanup |
| `qa t3-multi-pr/lock-conflict` | Two PRs same dir, verify lock conflict blocks second PR |
| `qa t3-multi-pr/lock-release-replan` | Full contention cycle: lock → conflict → release → replan |
| `qa all-t1` | Run all T1 scenarios sequentially |
| `qa all-t2` | Run all T2 scenarios sequentially |
| `qa cleanup` | Close test PRs, delete branches, release locks |
| `qa status` | Check tofuwok API, list locks and open test PRs |

### 4. Permissions

Claude Code will prompt you to approve each Bash command (git push, gh pr create, curl, etc.). This is your safety net — the agent won't ask "should I continue?" but you gate each shell command.

To run fully hands-off, pre-approve Bash in your Claude Code settings. For first runs, leave permissions on so you can watch.

### 5. Prerequisites

Before running scenarios:

- **Tofuwok backend running** with cloudflare tunnel active (`https://tofuwok.armhr.dev/health` returns "ok")
- **`terraform-orchestrator-gha` repo** has tofuwok workflows on main (`affected.yaml`, `tofuwok-plan.yaml`, `tofuwok-apply.yaml`)
- **Repo registered in tofuwok** with `execution_mode: "gha"` and GitHub App installed
- **`gh` CLI authenticated** with access to `jerny-stoiclane/terraform-orchestrator-gha`

The agent's preflight check verifies all of this before running any scenario.

### 6. Results

After each scenario, results are written to:

```
results/{scenario-name}-{YYYYMMDD-HHMMSS}.md
```

The agent prints a one-line summary per phase and a final pass/fail at the end.

## Configuration

```
TOFUWOK_API=https://tofuwok.armhr.dev
TOFUWOK_TOKEN=twk_86fe627e207a6b5d2d57702d290b85583d0ea0f4e994d9d350ba82980c206112
TARGET_REPO=jerny-stoiclane/terraform-orchestrator-gha
TARGET_REPO_PATH=/Users/jernz/stoiclane/tf-orchestrator-gha
OWNER=jerny-stoiclane
REPO=terraform-orchestrator-gha
```

These are hardcoded in the agent definition. To test against a different repo, edit `.claude/agents/qa.md`.

## Conventions

- Test branches: `test-qa/{run-id}`
- PR titles: `[qa] {scenario} — {run-id}`
- Results: `results/{scenario-name}-{timestamp}.md`
- Agent never closes PRs mid-scenario — only merges (to test apply) or leaves for cleanup
- Cleanup always runs even on failure
- Cleanup is the ONE place where test PRs are closed
