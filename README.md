# tofuwok-qa-agent

Autonomous E2E testing agent for [tofuwok](https://github.com/jerny-stoiclane/tofuwok) + [tf-orchestrator-gha](https://github.com/jerny-stoiclane/terraform-orchestrator-gha) integration.

Built as a [Claude Code agent](https://docs.anthropic.com/en/docs/claude-code/agents) that reads scenario specs, executes them against live infrastructure (real GitHub repo, real tofuwok API, real GitHub Actions workflows), and writes structured results.

## How it works

```
scenarios/*.md          Human-written test specs (intent, not implementation)
        |
  .claude/agents/qa.md  Agent definition — execution model, recovery, conventions
        |
    bin/twk              CLI wrapper for tofuwok API
    bin/qa-db            SQLite result tracker
    bin/tf-state         Terraform state verifier
        |
  results/*.md           Structured evidence reports per run
  bugs/*.md              Auto-filed bug reports with repro steps
```

The agent reads scenario files, decides how to execute them (which API calls, what order, how to recover from failures), and writes detailed evidence reports. It doesn't ask for help mid-scenario — it runs to completion or failure, then reports.

## Running

### Interactive (recommended for first runs)

```bash
claude
```

Then invoke the agent:

```
qa t1-smoke/api-health
```

Claude Code prompts you to approve each shell command. This is your safety net.

### Headless

```bash
claude -p "qa t2-single-pr/plan-apply-merge" --agent qa
```

### CI-friendly

```bash
claude --bare -p "qa t2-single-pr/plan-apply-merge" --agent qa --allowedTools "Read,Write,Bash,Glob,Grep"
```

## Scenarios

Scenarios are organized by tier (complexity/risk) and group (feature area):

| Tier | What | PRs | Duration |
|------|------|-----|----------|
| `t0-integration` | API endpoint + CLI checks | No | <30s |
| `t1-smoke` | Environment health | No | <30s |
| `t2-single-pr` | Core plan/apply/merge lifecycle | 1 | 2-5min |
| `t2-errors` | Error handling (bad HCL, missing provider) | 1 | 1-3min |
| `t2-lifecycle` | PR events (close, sync, rebase) | 1-2 | 2-5min |
| `t2-locks` | Lock acquire/release/force | 1 | 2-5min |
| `t3-multi-pr` | Multi-PR lock contention | 2+ | 5-10min |
| `t3-real-infra` | Real AWS resources (S3 state, OIDC, multi-region) | 1 | 5-10min |

### Available commands

| Command | What it does |
|---------|-------------|
| `qa {scenario}` | Run one scenario (e.g. `qa t2-single-pr/plan-apply-merge`) |
| `qa {group}` | Run all scenarios in a group (e.g. `qa t2-locks`) |
| `qa all-t1` | All T1 scenarios |
| `qa all-t2` | All T2 scenarios (safest-first order) |
| `qa cleanup` | Close test PRs, delete branches, release locks |
| `qa status` | Check tofuwok API, list locks and open test PRs |
| `qa results` | Recent test results from DB |
| `qa summary` | Pass/fail counts per scenario |
| `qa flaky` | Show flaky scenarios |
| `qa bugs` | List open bug reports |

See [scenarios/GUIDE.md](scenarios/GUIDE.md) for how to write new scenarios.

## Prerequisites

- **Tofuwok backend running** with tunnel active
- **tf-orchestrator-gha** repo has tofuwok workflows on main
- **Repo registered in tofuwok** with `execution_mode: "gha"`
- **`gh` CLI authenticated** with repo access

The agent's preflight check verifies all of this before running any scenario.

## Results and test history

Every scenario run produces two artifacts:

### Markdown report

Written to `results/{scenario}-{timestamp}.md` with full evidence:

- Every API call made and its key response fields
- Pass/fail per assertion with what was expected vs. received
- Timeline of events
- Summary with pass/fail counts

### SQLite database

All runs are recorded in `results/qa.db`. The agent records each run at start (`status=running`) and updates it on completion (`pass`, `fail`, or `invalid`). Query it with `bin/qa-db`:

```bash
bin/qa-db latest                        # Most recent run per scenario
bin/qa-db history --scenario plan-apply  # History for one scenario
bin/qa-db summary --days 7              # Pass/fail counts, last 7 days
bin/qa-db flaky --days 7                # Scenarios with mixed pass/fail
bin/qa-db regression                    # Scenarios that newly started failing
bin/qa-db fixes                         # Scenarios that newly started passing
bin/qa-db streak --min 3                # 3+ consecutive failures
bin/qa-db blame <scenario>              # First run in the current fail streak
bin/qa-db health --window 10            # TestGrid-style status per scenario
bin/qa-db running                       # In-progress test runs
```

The DB schema tracks: run ID, scenario name, status, duration, assertion counts, PR number, HEAD SHA, notes, and results file path.

## Project structure

```
.claude/
  agents/qa.md       Agent definition (execution model, API reference, recovery)
bin/
  twk                CLI for tofuwok API (auth, polling, field selection)
  qa-db              SQLite-backed result tracker
  tf-state           Terraform state verification
scenarios/
  GUIDE.md           How to write scenarios
  BACKLOG.md         Edge cases identified, prioritized
  t0-integration/    API + CLI checks
  t1-smoke/          Health checks
  t2-*/              Single-PR test flows
  t3-*/              Multi-PR and real infra tests
results/             Test run reports (gitignored)
bugs/                Auto-filed bug reports with repro steps
CLAUDE.md            Agent behavior conventions
```

## Conventions

- Test branches: `test-qa/{run-id}`
- PR titles: `[qa] {scenario} — {run-id}`
- The agent merges PRs to test apply — it never closes mid-scenario
- Cleanup always runs, even on failure
- All tofuwok API calls go through `bin/twk`, never raw curl
