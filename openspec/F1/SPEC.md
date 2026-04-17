# F1: Autonomous QA Agent for Tofuwok + TF Orchestrator

**Status:** Draft
**Date:** 2026-04-16

---

## Intent

Build a Claude Code agent that runs end-to-end test scenarios against the tofuwok backend and `tf-orchestrator-gha` repo without getting stuck or requiring human intervention mid-scenario. The agent operates as a bash-driven state machine that uses Claude for decision-making on failures, not for step-by-step conversation.

The existing tofuwok QA agent (`~/stoiclane/tofuwok/.claude/agents/tofuwok-qa.md`) has the right scenario structure but gets stuck on async waits, requires too much approval gating, and can't recover from transient failures (502s, stuck runs, stale locks).

---

## Scope

### In Scope

- Agent definition (`.claude/agents/qa.md`) that runs locally
- Bash helper library (`lib/`) for polling, retries, state tracking, recovery
- Scenario definitions for tofuwok runner model (affected → dispatch → plan → apply)
- Scenarios: smoke, single-PR lifecycle, multi-PR lock contention
- Self-recovery from transient failures (502, stuck runs, orphan locks)
- Structured result output (JSON + markdown)
- Automatic bug filing on assertion failure

### Out of Scope

- GitHub Actions agent (runs locally only)
- Agent SDK / remote execution
- UI testing (browser-based)
- Real AWS infra testing (null_resource only)
- Drift detection testing (separate feature)

---

## Architecture

### State Machine, Not Conversation

Each scenario is a sequence of states. The agent reads the scenario, builds the state list, and executes each state. If a state fails transiently, it retries. If it fails permanently, it records the failure and moves to cleanup.

```
SCENARIO FILE (markdown)
  → PARSE (extract states + assertions)
  → EXECUTE (state machine)
  → REPORT (structured results)
```

### State Execution Model

```
┌─────────┐     ┌──────────┐     ┌────────┐     ┌─────────┐
│ TRIGGER  │────→│   WAIT   │────→│ VERIFY │────→│  NEXT   │
│ (action) │     │ (poll)   │     │(assert)│     │ (state) │
└─────────┘     └──────────┘     └────────┘     └─────────┘
                     │                │
                     ▼                ▼
                ┌──────────┐    ┌──────────┐
                │  RETRY   │    │   BUG    │
                │(recover) │    │ (file)   │
                └──────────┘    └──────────┘
```

Each state has:
- **action**: what to do (bash command or API call)
- **wait**: how to know it's done (poll function + timeout)
- **verify**: assertions to check after completion
- **recover**: what to do if the wait times out or the action fails
- **max_retries**: how many times to retry before failing (default: 3)

### Helper Library (`lib/`)

Reusable bash functions that handle the hard parts:

#### `lib/wait.sh` — Async Polling

```bash
# wait_for_run: poll tofuwok until run reaches terminal state
# Returns: JSON of completed run, or exits 1 on timeout
# Recovery: if stuck >2x expected duration, cancel and retrigger
wait_for_run() {
  local owner=$1 repo=$2 pr=$3 dir=$4 run_type=$5
  local timeout=${6:-300} interval=${7:-10}
  local elapsed=0

  while [ $elapsed -lt $timeout ]; do
    local run=$(curl -sf "$TOFUWOK_API/api/v1/runs/$owner/$repo" \
      | jq -r "[.[] | select(.pr_number==$pr and .dir==\"$dir\" and .run_type==\"$run_type\")] | sort_by(.started_at) | last")
    local status=$(echo "$run" | jq -r '.status // "unknown"')

    case $status in
      success|failure|cancelled)
        echo "$run"
        return 0
        ;;
      running|pending)
        sleep $interval
        elapsed=$((elapsed + interval))
        ;;
      *)
        sleep $interval
        elapsed=$((elapsed + interval))
        ;;
    esac
  done

  echo '{"error":"timeout","elapsed":'$elapsed'}' >&2
  return 1
}
```

#### `lib/gh.sh` — GitHub Operations with Recovery

```bash
# create_test_pr: create a PR with retry on failure
# Handles: rate limits (429), server errors (500+), network errors
create_test_pr() {
  local repo=$1 branch=$2 title=$3 body=$4
  local max_retries=3

  for attempt in $(seq 1 $max_retries); do
    local result=$(gh pr create --repo "$repo" \
      --head "$branch" --base main \
      --title "$title" --body "$body" 2>&1)
    
    if echo "$result" | grep -q "https://github.com"; then
      local pr_num=$(echo "$result" | grep -oE '/pull/[0-9]+' | grep -oE '[0-9]+')
      echo "$pr_num"
      return 0
    fi

    echo "Attempt $attempt failed: $result" >&2
    sleep $((attempt * 5))
  done

  return 1
}

# wait_for_workflow: wait for a GHA workflow to complete
# Uses gh run list with polling, not sleep loops
wait_for_workflow() {
  local repo=$1 workflow=$2 branch=$3 timeout=${4:-300}
  local elapsed=0 interval=15

  while [ $elapsed -lt $timeout ]; do
    local run=$(gh run list --repo "$repo" --workflow="$workflow" \
      --branch="$branch" --limit 1 \
      --json databaseId,status,conclusion 2>/dev/null \
      | jq '.[0] // empty')
    
    local status=$(echo "$run" | jq -r '.status // "unknown"')
    
    if [ "$status" = "completed" ]; then
      echo "$run"
      return 0
    fi

    sleep $interval
    elapsed=$((elapsed + interval))
  done

  return 1
}
```

#### `lib/tofuwok.sh` — API Helpers with Recovery

```bash
# tofuwok_api: curl wrapper with retry on 502/503
tofuwok_api() {
  local method=$1 path=$2 body=$3
  local max_retries=3

  for attempt in $(seq 1 $max_retries); do
    local http_code response
    if [ -n "$body" ]; then
      response=$(curl -sf -w '\n%{http_code}' -X "$method" \
        -H "Authorization: Bearer $TOFUWOK_TOKEN" \
        -H "Content-Type: application/json" \
        "$TOFUWOK_API$path" -d "$body" 2>/dev/null)
    else
      response=$(curl -sf -w '\n%{http_code}' -X "$method" \
        -H "Authorization: Bearer $TOFUWOK_TOKEN" \
        "$TOFUWOK_API$path" 2>/dev/null)
    fi

    http_code=$(echo "$response" | tail -1)
    body_out=$(echo "$response" | sed '$d')

    case $http_code in
      2*) echo "$body_out"; return 0 ;;
      502|503) echo "Retry $attempt: $http_code" >&2; sleep $((attempt * 3)) ;;
      409) echo "$body_out"; return 2 ;; # lock conflict, not an error
      *) echo "$body_out"; return 1 ;;
    esac
  done

  return 1
}

# cleanup_locks: force-release all locks for a repo
cleanup_locks() {
  local owner=$1 repo=$2
  local locks=$(tofuwok_api GET "/api/v1/locks/$owner/$repo")
  echo "$locks" | jq -c '.[]' | while read -r lock; do
    local dir=$(echo "$lock" | jq -r '.dir')
    local dir_enc=$(printf '%s' "$dir" | jq -sRr @uri)
    tofuwok_api DELETE "/api/v1/locks/$owner/$repo?dir=$dir_enc&workspace=default&released_by=qa-agent"
    echo "Released lock: $dir"
  done
}
```

#### `lib/state.sh` — Scenario State Tracking

```bash
# State is a JSON file: /tmp/qa-{scenario}-{run_id}.json
init_state() {
  local scenario=$1 run_id=$2
  STATE_FILE="/tmp/qa-${scenario}-${run_id}.json"
  echo '{"scenario":"'$scenario'","run_id":"'$run_id'","states":[],"current":0,"status":"running"}' > "$STATE_FILE"
}

record_state() {
  local name=$1 status=$2 output=$3
  local tmp=$(mktemp)
  jq --arg name "$name" --arg status "$status" --arg output "$output" --arg ts "$(date -Iseconds)" \
    '.states += [{"name":$name,"status":$status,"output":$output,"completed_at":$ts}] | .current += 1' \
    "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
}

get_state() {
  cat "$STATE_FILE"
}
```

---

## Scenario Format

Each scenario is a markdown file with structured frontmatter and a state list:

```markdown
# Scenario: Single PR Plan + Apply + Merge

tier: t2
timeout: 300
target_repo: jerny-stoiclane/terraform-orchestrator-gha
dirs:
  - test/companies/bravo/snowflake

## States

### 1. create-branch
action: |
  git checkout main && git pull
  git checkout -b test-qa/{{RUN_ID}}
  echo '# {{RUN_ID}}' >> test/companies/bravo/snowflake/variables.tf
  git add -A && git commit -m "test: {{RUN_ID}}"
  git push -u origin test-qa/{{RUN_ID}}
cleanup: git push origin --delete test-qa/{{RUN_ID}}

### 2. create-pr
action: |
  gh pr create --repo {{REPO}} --head test-qa/{{RUN_ID}} --base main \
    --title "[qa] {{SCENARIO}} — {{RUN_ID}}" \
    --body "Automated QA test"
output: PR_NUMBER

### 3. wait-affected
wait: workflow_complete
workflow: affected.yaml
branch: test-qa/{{RUN_ID}}
timeout: 60

### 4. wait-plan
wait: tofuwok_run
run_type: plan
pr: {{PR_NUMBER}}
dir: test/companies/bravo/snowflake
timeout: 180

### 5. verify-plan
assert:
  - run.status == "success"
  - run.has_changes == true
  - lock.pr_number == {{PR_NUMBER}}
  - commit_status("tofuwok/plan/test/companies/bravo/snowflake") == "success"

### 6. merge-pr
action: gh pr merge {{PR_NUMBER}} --repo {{REPO}} --merge

### 7. wait-apply
wait: tofuwok_run
run_type: apply
pr: {{PR_NUMBER}}
dir: test/companies/bravo/snowflake
timeout: 180

### 8. verify-apply
assert:
  - run.status == "success"
  - lock.applied == true
  - commit_status("tofuwok/apply/test/companies/bravo/snowflake") == "success"

### 9. verify-cleanup
assert:
  - lock == null  # released after merge+apply
  - pr.state == "closed"

## Cleanup
- git push origin --delete test-qa/{{RUN_ID}}
```

---

## Agent Definition

### `.claude/agents/qa.md`

The agent:
1. Reads scenario files from `scenarios/`
2. Sources `lib/*.sh` for helpers
3. Parses scenario into state list
4. Executes each state with retry/recovery
5. Writes results to `results/`
6. Never asks user mid-scenario — runs to completion or failure

### Key Behaviors

**Autonomous execution:**
- Given a scenario name, run it end-to-end
- On transient failure (502, timeout): retry with backoff (max 3)
- On permanent failure (assertion mismatch): record bug, continue to cleanup
- On stuck state (>2x timeout): force cancel/release, skip to cleanup

**Cleanup always runs:**
- Delete test branches
- Close test PRs (if not merged)
- Release orphan locks
- Cleanup runs even if scenario failed

**No conversation mid-scenario:**
- Agent outputs progress lines, not questions
- If something unexpected happens, log it and continue
- Only ask user when: choosing which scenarios to run, or confirming T3+ tier

**Structured output:**
```json
{
  "scenario": "single-pr-plan-apply",
  "run_id": "20260416-234500-a1b2",
  "status": "pass",
  "duration_s": 185,
  "states": [
    {"name": "create-branch", "status": "pass", "duration_s": 3},
    {"name": "wait-plan", "status": "pass", "duration_s": 95}
  ],
  "assertions": {
    "total": 8,
    "passed": 8,
    "failed": 0
  },
  "bugs": []
}
```

---

## Scenarios

### T1: Smoke

| Scenario | What it tests |
|---|---|
| `api-health` | Tofuwok API responds, repo registered, execution_mode=gha |
| `gh-access` | gh CLI can read/write to target repo |

### T2: Single PR

| Scenario | What it tests |
|---|---|
| `single-dir-plan` | One dir changed → affected detected → plan dispatched → plan succeeds → lock acquired |
| `multi-dir-plan` | All dirs changed → 6 plans dispatched → all succeed → all locks acquired |
| `plan-apply-merge` | Plan → merge → apply dispatched → apply succeeds → lock released |
| `plan-failure` | Bad terraform → plan fails → error surfaced in PR comment |
| `stale-plan-replan` | Push new commit → old plan superseded → new plan runs |

### T3: Multi-PR Lock Contention

| Scenario | What it tests |
|---|---|
| `lock-conflict` | PR #1 locks dir A → PR #2 touches dir A → PR #2 plan sees 409 → PR #2 merge blocked |
| `lock-release-replan` | PR #1 merges+applies → lock released → PR #2 branch updated → PR #2 re-plans → lock acquired |
| `overlapping-dirs` | PR #1 touches A+B, PR #2 touches B+C → B is contested, A and C are free |

### T4: Edge Cases

| Scenario | What it tests |
|---|---|
| `backend-502-recovery` | Tofuwok goes down during plan → plan retried after recovery |
| `orphan-lock-cleanup` | PR closed without merge → locks released |
| `rapid-push` | 3 pushes in 10 seconds → only latest plan runs (supersession) |

---

## Configuration

```bash
# env vars the agent needs
TOFUWOK_API=https://tofuwok.armhr.dev
TOFUWOK_TOKEN=twk_...
TARGET_REPO=jerny-stoiclane/terraform-orchestrator-gha
TARGET_REPO_PATH=~/stoiclane/tf-orchestrator-gha
QA_BRANCH_PREFIX=test-qa/
```

---

## Files

| File | Purpose |
|------|---------|
| `.claude/agents/qa.md` | Agent definition |
| `lib/wait.sh` | Polling + timeout helpers |
| `lib/gh.sh` | GitHub CLI wrappers with retry |
| `lib/tofuwok.sh` | Tofuwok API helpers with recovery |
| `lib/state.sh` | Scenario state tracking |
| `lib/assert.sh` | Assertion helpers (check run status, lock state, commit statuses) |
| `scenarios/t1-smoke/*.md` | Smoke test scenarios |
| `scenarios/t2-single-pr/*.md` | Single PR lifecycle scenarios |
| `scenarios/t3-multi-pr/*.md` | Multi-PR lock contention scenarios |
| `scenarios/t4-edge-cases/*.md` | Edge case scenarios |
| `results/` | Test results (JSON + markdown) |
| `CLAUDE.md` | Project context for the agent |

---

## Acceptance Criteria

- [ ] Agent runs `t1-smoke/api-health` without human intervention
- [ ] Agent runs `t2-single-pr/single-dir-plan` end-to-end: create PR → wait for plan → verify → cleanup
- [ ] Agent recovers from tofuwok 502 without asking user
- [ ] Agent recovers from stuck run (cancel + retrigger) without asking user
- [ ] Agent cleans up test branches and PRs even on failure
- [ ] Agent writes structured JSON results for every scenario
- [ ] Agent files bugs (markdown in results/) on assertion failure
- [ ] T3 lock contention scenario: 2 PRs, lock conflict detected, merge blocked, release + replan works
- [ ] Agent completes a full T1+T2 run in under 15 minutes
