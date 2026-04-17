# F1: Autonomous QA Agent for Tofuwok + TF Orchestrator

**Status:** Draft
**Date:** 2026-04-16

---

## Intent

Build a Claude Code agent that runs E2E test scenarios against the tofuwok backend and `tf-orchestrator-gha` repo autonomously. The agent follows the pattern used by dagster's buildkite-expert (tiered response, smart escalation) and tofuwok's developer agent (read spec → execute → report) — but for QA workflows that involve creating PRs, waiting for GHA dispatches, verifying tofuwok API state, and testing lock contention.

The existing tofuwok QA agent gets stuck because: too many approval gates, inline polling that times out, no self-recovery. This agent fixes those problems.

---

## Scope

### In Scope

- Agent definition (`.claude/agents/qa.md`)
- Inline skills for: preflight, wait-for-dispatch, wait-for-run, verify-plan, verify-apply, verify-lock, cleanup, recover
- Scenario definitions for the tofuwok runner model
- T1 (smoke), T2 (single PR lifecycle), T3 (multi-PR lock contention)
- Self-recovery from transient failures
- Structured result output

### Out of Scope

- GitHub Actions agent (local only)
- UI testing
- Real AWS infra (null_resource only)
- Agent SDK / remote execution

---

## Agent Design

### Core Principle: Execute, Don't Converse

The agent reads a scenario, executes it front-to-back, and reports results. It does NOT:
- Ask for approval between steps (T1/T2 are auto-approved)
- Wait for human input during async operations
- Stop and ask "should I continue?" on transient failures

It DOES:
- Report progress after each major phase
- Self-recover from known failure modes (502, stuck runs, orphan locks)
- Stop and report clearly on unknown/permanent failures
- Always run cleanup, even on failure

### Three-Tier Response (from dagster buildkite-expert pattern)

| Tier | Trigger | Autonomy | Duration |
|------|---------|----------|----------|
| T1: Smoke | "run smoke" | Fully auto, no PRs | 30s |
| T2: Single PR | "run {scenario}" | Fully auto, creates 1 PR | 3-5min |
| T3: Multi PR | "run lock-contention" | Auto, creates 2+ PRs | 5-10min |

No approval gates for any tier. Agent runs to completion or failure.

### Startup Sequence (from tofuwok developer pattern)

When invoked with a scenario name:

```
1. PREFLIGHT
   - Verify tofuwok API: curl $TOFUWOK_API/health
   - Verify gh CLI: gh auth status
   - Verify target repo: gh repo view $TARGET_REPO
   - Check for stale test branches: gh pr list --search "test-qa/"
   - Clean stale locks: curl $TOFUWOK_API/api/v1/locks/$OWNER/$REPO
   → If any check fails: report and STOP (don't guess)

2. READ SCENARIO
   - Read scenarios/{tier}/{name}.md
   - Parse into ordered state list
   → Report: "Running {scenario} — {N} states, estimated {M}s"

3. EXECUTE (see Skills below)

4. REPORT
   - Write results to results/{scenario}-{timestamp}.md
   - Print summary: PASS/FAIL, assertions, duration, bugs
```

---

## Skills (Inline Procedures)

Skills are documented procedures the agent knows how to execute. Not bash scripts — the agent reads these and uses Bash tool calls.

### skill:preflight

```
PURPOSE: Verify environment before any scenario

CHECK tofuwok API health:
  curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/health"
  FAIL if not "ok"

CHECK tofuwok repo registered:
  curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/repos" | jq '.[] | select(.repo=="terraform-orchestrator-gha")'
  FAIL if not found or execution_mode != "gha"

CHECK gh CLI:
  gh auth status
  gh repo view $TARGET_REPO --json name
  FAIL if either fails

CHECK no stale test artifacts:
  STALE_PRS=$(gh pr list --repo $TARGET_REPO --search "test-qa/" --state open --json number)
  STALE_LOCKS=$(curl -sf "$TOFUWOK_API/api/v1/locks/$OWNER/$REPO" | jq length)
  If stale PRs or locks exist: run skill:cleanup first, then continue

REPORT: "Preflight PASS — tofuwok up, repo registered, gh authenticated"
```

### skill:create-test-branch

```
PURPOSE: Create a branch with terraform changes for testing

INPUTS: dirs (list of dirs to touch), run_id (unique identifier)

STEPS:
  cd $TARGET_REPO_PATH
  git checkout main && git pull origin main
  git checkout -b test-qa/$RUN_ID
  for each dir in dirs:
    echo "# $RUN_ID" >> $dir/variables.tf
  git add -A
  git commit -m "test: $RUN_ID"
  git push -u origin test-qa/$RUN_ID

OUTPUT: branch name
CLEANUP: git push origin --delete test-qa/$RUN_ID
```

### skill:create-test-pr

```
PURPOSE: Open a PR and wait for the affected workflow to complete

INPUTS: branch, title

STEPS:
  PR_NUM=$(gh pr create --repo $TARGET_REPO --head $branch --base main --title "$title" --body "QA test $RUN_ID" | grep -oE '/pull/[0-9]+' | grep -oE '[0-9]+')
  
  WAIT for "Tofuwok" workflow to complete (affected detection):
    TIMEOUT: 90s, INTERVAL: 10s
    POLL: gh run list --repo $TARGET_REPO --branch $branch --workflow affected.yaml --limit 1 --json status,conclusion
    DONE when: status == "completed"
    RECOVERY on timeout: check gh run list for errors, report and continue

OUTPUT: PR_NUM
```

### skill:wait-for-plans

```
PURPOSE: Wait for tofuwok to dispatch and complete plan workflows

INPUTS: pr_number, expected_dirs (list)

STEPS:
  TIMEOUT: 300s, INTERVAL: 15s
  
  POLL tofuwok runs API:
    RUNS=$(curl -sf "$TOFUWOK_API/api/v1/runs/$OWNER/$REPO" -H "Authorization: Bearer $TOFUWOK_TOKEN")
    PLAN_RUNS=$(echo $RUNS | jq "[.[] | select(.pr_number==$PR_NUM and .run_type==\"plan\")]")
    
    CHECK each expected dir has a terminal run (success/failure):
      COMPLETED=$(echo $PLAN_RUNS | jq "[.[] | select(.status==\"success\" or .status==\"failure\")] | length")
      EXPECTED=${#expected_dirs[@]}
    
    DONE when: COMPLETED >= EXPECTED

  RECOVERY on timeout:
    Check which dirs are missing/stuck
    For stuck runs (status=running for >2x timeout): 
      Cancel via POST $TOFUWOK_API/api/v1/runs/{id}/cancel
      Retrigger via POST $TOFUWOK_API/api/v1/trigger
    Retry wait once

OUTPUT: JSON array of completed plan runs
```

### skill:wait-for-applies

```
PURPOSE: Wait for tofuwok to dispatch and complete apply workflows after merge

INPUTS: pr_number, expected_dirs (list)

Same pattern as skill:wait-for-plans but filters for run_type=="apply"

TIMEOUT: 300s (applies take longer)
```

### skill:verify-locks

```
PURPOSE: Verify lock state matches expectations

INPUTS: expected_locks (list of {dir, pr_number, applied})

STEPS:
  LOCKS=$(curl -sf "$TOFUWOK_API/api/v1/locks/$OWNER/$REPO" -H "Authorization: Bearer $TOFUWOK_TOKEN")
  
  For each expected lock:
    LOCK=$(echo $LOCKS | jq ".[] | select(.dir==\"$dir\")")
    ASSERT lock exists (or doesn't, if expected null)
    ASSERT lock.pr_number == expected
    ASSERT lock.applied == expected
    ASSERT lock.lock_policy == "strict"

OUTPUT: list of assertion results
```

### skill:verify-commit-statuses

```
PURPOSE: Verify GitHub commit statuses match expectations

INPUTS: sha, expected_statuses (list of {context, state})

STEPS:
  STATUSES=$(gh api repos/$TARGET_REPO/commits/$sha/statuses)
  
  For each expected status:
    ACTUAL=$(echo $STATUSES | jq -r ".[] | select(.context==\"$context\") | .state" | head -1)
    ASSERT actual == expected

OUTPUT: list of assertion results
```

### skill:verify-pr-comment

```
PURPOSE: Verify tofuwok posted a PR comment with expected content

INPUTS: pr_number, expected_content (list of strings)

STEPS:
  COMMENTS=$(gh api repos/$TARGET_REPO/issues/$pr_number/comments)
  LATEST=$(echo $COMMENTS | jq -r '.[-1].body')
  
  For each expected string:
    ASSERT LATEST contains string

OUTPUT: assertion results
```

### skill:merge-pr

```
PURPOSE: Merge a PR and wait for apply dispatch

INPUTS: pr_number

STEPS:
  gh pr merge $pr_number --repo $TARGET_REPO --merge
  
  WAIT for "Tofuwok" workflow to trigger on merge (closed event):
    TIMEOUT: 60s
    POLL: gh run list --repo $TARGET_REPO --workflow affected.yaml --limit 1 --json status
    DONE when: new run appears and completes

OUTPUT: merge SHA
```

### skill:cleanup

```
PURPOSE: Clean up all test artifacts. ALWAYS runs, even on failure.

STEPS:
  # Close open test PRs
  TEST_PRS=$(gh pr list --repo $TARGET_REPO --search "test-qa/" --state open --json number --jq '.[].number')
  for pr in $TEST_PRS:
    gh pr close $pr --repo $TARGET_REPO --delete-branch

  # Delete remote test branches
  BRANCHES=$(git -C $TARGET_REPO_PATH branch -r | grep "origin/test-qa/" | sed 's|origin/||')
  for branch in $BRANCHES:
    git -C $TARGET_REPO_PATH push origin --delete $branch

  # Release orphan locks
  LOCKS=$(curl -sf "$TOFUWOK_API/api/v1/locks/$OWNER/$REPO" -H "Authorization: Bearer $TOFUWOK_TOKEN")
  echo $LOCKS | jq -c '.[]' | while read lock:
    DIR=$(echo $lock | jq -r '.dir')
    DIR_ENC=$(printf '%s' "$DIR" | jq -sRr @uri)
    curl -sf -X DELETE "$TOFUWOK_API/api/v1/locks/$OWNER/$REPO?dir=$DIR_ENC&workspace=default&released_by=qa-cleanup" -H "Authorization: Bearer $TOFUWOK_TOKEN"

  # Return to main
  cd $TARGET_REPO_PATH && git checkout main
```

### skill:recover-stuck-run

```
PURPOSE: Recover from a run stuck in "running" state

INPUTS: run_id

STEPS:
  # Check if GHA workflow is actually running
  GHA_RUNS=$(gh run list --repo $TARGET_REPO --limit 20 --json databaseId,status,name)
  
  # If no matching GHA workflow is running, the tofuwok run is orphaned
  # Cancel it so tofuwok allows retrigger
  curl -sf -X POST "$TOFUWOK_API/api/v1/runs/$run_id/cancel" -H "Authorization: Bearer $TOFUWOK_TOKEN"
  
  # Wait 5s for cancellation to propagate
  sleep 5

OUTPUT: cancelled run ID
```

---

## Scenario Format

Scenarios are markdown with structured sections. The agent parses them.

```markdown
---
name: single-dir-plan-apply-merge
tier: t2
timeout: 300
target_repo: jerny-stoiclane/terraform-orchestrator-gha
---

# Single Dir: Plan → Apply → Merge

## Setup
dirs:
  - test/companies/bravo/snowflake

## Phases

### Phase 1: Create PR
skill: create-test-branch
  dirs: [test/companies/bravo/snowflake]
skill: create-test-pr
  title: "[qa] single dir plan+apply"
→ captures: PR_NUMBER, BRANCH, HEAD_SHA

### Phase 2: Verify Plan
skill: wait-for-plans
  pr_number: {{PR_NUMBER}}
  expected_dirs: [test/companies/bravo/snowflake]
skill: verify-locks
  expected: [{dir: "test/companies/bravo/snowflake", pr_number: {{PR_NUMBER}}, applied: false}]
skill: verify-commit-statuses
  sha: {{HEAD_SHA}}
  expected: [{context: "tofuwok/plan/test/companies/bravo/snowflake", state: "success"}]

### Phase 3: Merge + Apply
skill: merge-pr
  pr_number: {{PR_NUMBER}}
→ captures: MERGE_SHA
skill: wait-for-applies
  pr_number: {{PR_NUMBER}}
  expected_dirs: [test/companies/bravo/snowflake]
skill: verify-locks
  expected: [{dir: "test/companies/bravo/snowflake", pr_number: null}]  # released after apply+merge

### Cleanup
skill: cleanup
```

---

## Scenarios

### T1: Smoke

| Scenario | File | What |
|---|---|---|
| `api-health` | `t1-smoke/api-health.md` | Tofuwok healthy, repo registered, execution_mode=gha |
| `gh-access` | `t1-smoke/gh-access.md` | gh can read/write target repo, list workflows |
| `clean-state` | `t1-smoke/clean-state.md` | No stale locks, no open test PRs |

### T2: Single PR Lifecycle

| Scenario | File | What |
|---|---|---|
| `single-dir-plan` | `t2-single-pr/single-dir-plan.md` | 1 dir → affected → plan dispatched → plan succeeds → lock acquired |
| `multi-dir-plan` | `t2-single-pr/multi-dir-plan.md` | 6 dirs → 6 plans dispatched → all succeed → 6 locks |
| `plan-apply-merge` | `t2-single-pr/plan-apply-merge.md` | Plan → merge → apply → lock released |
| `full-cycle` | `t2-single-pr/full-cycle.md` | Plan → merge → apply → verify comment + status + lock cleanup |

### T3: Multi-PR Lock Contention

| Scenario | File | What |
|---|---|---|
| `lock-conflict` | `t3-multi-pr/lock-conflict.md` | PR1 locks dir → PR2 plan sees conflict → PR2 lock blocked |
| `lock-release-replan` | `t3-multi-pr/lock-release-replan.md` | PR1 merge+apply → lock released → PR2 re-plans → PR2 lock acquired |
| `overlapping-dirs` | `t3-multi-pr/overlapping-dirs.md` | PR1 has A+B, PR2 has B+C → B contested, A+C free |

---

## Configuration

```
# .env or CLAUDE.md
TOFUWOK_API=https://tofuwok.armhr.dev
TOFUWOK_TOKEN=twk_...
TARGET_REPO=jerny-stoiclane/terraform-orchestrator-gha
TARGET_REPO_PATH=~/stoiclane/tf-orchestrator-gha
OWNER=jerny-stoiclane
REPO=terraform-orchestrator-gha
```

---

## Files

| File | Purpose |
|------|---------|
| `.claude/agents/qa.md` | Agent definition with inline skills |
| `CLAUDE.md` | Project context, config, conventions |
| `scenarios/t1-smoke/*.md` | Smoke scenarios |
| `scenarios/t2-single-pr/*.md` | Single PR scenarios |
| `scenarios/t3-multi-pr/*.md` | Multi-PR scenarios |
| `results/` | Test results (markdown) |

---

## Acceptance Criteria

- [ ] `qa run t1-smoke/api-health` completes without human intervention
- [ ] `qa run t2-single-pr/single-dir-plan` creates PR, waits for plan, verifies lock, cleans up — no questions asked
- [ ] `qa run t2-single-pr/plan-apply-merge` does full cycle: PR → plan → merge → apply → verify → cleanup
- [ ] Agent recovers from tofuwok 502 (retry 3x with backoff)
- [ ] Agent recovers from stuck tofuwok run (cancel + retrigger)
- [ ] Agent cleans up test branches, PRs, and locks even on failure
- [ ] Agent produces markdown result file with pass/fail per assertion
- [ ] T3 lock contention: 2 PRs, conflict detected, block verified, release + replan works
- [ ] Full T1+T2 suite completes in under 10 minutes
