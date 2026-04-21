# Tofuwok QA Agent

Autonomous E2E testing agent for tofuwok + tf-orchestrator-gha. See [README.md](README.md) for project overview and usage.

## Agent

The QA agent lives at `.claude/agents/qa.md`. It is the core of this repo — the agent definition contains the full execution model, API reference, recovery procedures, and result format.

Invoke it with `qa {scenario}` in a Claude Code session, or headless via `claude -p "qa ..." --agent qa`.

## Configuration

Environment is hardcoded in the agent definition (`.claude/agents/qa.md`). To test against a different repo, edit the Environment table there.

## Conventions

- Test branches: `test-qa/{run-id}`
- PR titles: `[qa] {scenario} — {run-id}`
- Results: `results/{scenario-name}-{timestamp}.md`
- Agent merges PRs to test apply — never closes mid-scenario
- Cleanup always runs, even on failure
- All tofuwok API calls go through `bin/twk`, never raw curl
- Do not commit results files — the user decides when to commit
- Bug reports go to `bugs/BUG-{NNN}-{slug}.md` — only file with high confidence

## Writing scenarios

See [scenarios/GUIDE.md](scenarios/GUIDE.md). Scenarios describe intent, not implementation — the agent decides how to execute them.

## AWS resources

- Always use `--profile stoic-ai-test` for AWS CLI commands
- Never delete AWS resources via CLI — all destruction goes through terraform destroy
- All QA resources must be tagged with `ManagedBy=tofuwok-qa`
- Backend must be S3 (not local) — local state is lost between GHA runs
