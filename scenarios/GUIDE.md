# Writing QA Scenarios

## Format

Each scenario is a markdown file in `scenarios/{tier}-{group}/{name}.md`.

### Frontmatter

```yaml
---
name: descriptive-name
tier: t0|t1|t2|t3
timeout: 300
dirs:                              # optional — which TF dirs this test touches
  - test/companies/bravo/snowflake
---
```

### Structure

```markdown
# Title: What This Tests

One sentence explaining what this scenario verifies and why it matters.

## Phases

### Phase 1: {Action Name}
Describe what to do, not how to do it.
If there's a specific command, include it.
Captures: VARIABLE_NAME (values the agent needs for later phases)

### Phase 2: {Wait/Verify Name}
Describe what to wait for or verify.
Say what the expected outcome is.

### Phase 3: {Another Phase}
...

### Cleanup
What to clean up. The agent always runs this even on failure.
```

## Rules

### Describe intent, not implementation
```
# GOOD — says what to verify
Assert: plan run exists with status=success and has_changes=true

# BAD — prescribes exact bash
Assert: curl -sf ... | jq '.[] | select(.status=="success")' | ...
```

The agent knows `bin/twk`, the tofuwok API, `gh` CLI, and `agent-browser`. Let it choose the tool. Include specific commands only when the exact invocation matters (e.g., triggering apply with a specific SHA).

### Each phase should be one thing
- Create a PR (one phase)
- Wait for plan (one phase)
- Verify plan results (one phase)
- Trigger apply (one phase)

Don't combine "create PR and wait for plan" into one phase. The agent reports per-phase, so granularity = better diagnostics.

### Captures pass state between phases
```markdown
### Phase 1: Create PR
...
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Poll tofuwok for plan runs matching PR_NUMBER.
```

The agent reads "Captures:" and knows to save those values for later phases.

### Assert format
State what should be true. The agent verifies and records evidence.

```markdown
Assert:
- Plan run exists with status=success for this PR and dir
- has_changes == true
- Lock acquired by PR_NUMBER with applied=false
- Commit status tofuwok/plan set to success on HEAD_SHA
```

Each assert is independently verified. A failed assert doesn't skip the rest.

### Cleanup is mandatory
Every T2+ scenario must have a Cleanup section. The agent runs it even on failure. Describe what to clean up:

```markdown
### Cleanup
- Delete test branch
- Release locks if any remain
- For failed scenarios: close PR (don't merge bad state)
```

## Tiers

| Tier | What | PRs? | Duration |
|------|------|------|----------|
| t0-integration | API endpoint checks | No | <30s |
| t1-smoke | Environment health | No | <30s |
| t1-ui | Dashboard renders | No | <30s |
| t2-single-pr | One PR lifecycle | 1 | 2-5min |
| t2-errors | Plan failures | 1 (close, don't merge) | 1-3min |
| t2-lifecycle | PR events (close, sync, rebase) | 1-2 | 2-5min |
| t2-locks | Lock acquire/release/force | 1 | 2-5min |
| t3-multi-pr | Lock contention, concurrent PRs | 2+ | 5-10min |

## Groups

Put related scenarios together:

```
scenarios/
  t0-integration/    API endpoint + CLI verification
  t1-smoke/          Environment health checks
  t2-single-pr/      Core plan/apply/merge flows
  t2-errors/         Error handling (bad HCL, missing provider)
  t2-lifecycle/      PR events (close, synchronize, base update)
  t2-locks/          Lock lifecycle (acquire, release, force)
  t3-multi-pr/       Multi-PR lock contention
```

## Adding a New Scenario

1. Pick the right tier and group
2. Create `scenarios/{tier}-{group}/{name}.md`
3. Write frontmatter + phases + cleanup
4. Run it: `qa {tier}-{group}/{name}`
5. If it passes, the agent auto-chains to the next scenario in the group

## Example: Minimal T2 Scenario

```markdown
---
name: example-test
tier: t2
timeout: 180
dirs:
  - test/companies/bravo/snowflake
---

# Example: Add a Resource and Verify Plan

Verify that adding a null_resource produces a plan with has_changes=true.

## Phases

### Phase 1: Create Branch + PR
Create branch from main, add a null_resource to bravo/snowflake, push, open PR.
Captures: PR_NUMBER, HEAD_SHA

### Phase 2: Wait for Plan
Wait for tofuwok to dispatch and complete a plan run for this PR.
Timeout: 180s

### Phase 3: Verify Plan
Assert:
- Plan run exists with status=success
- has_changes == true
- resources_to_add >= 1
- Lock acquired by PR_NUMBER

### Phase 4: Apply + Merge
Trigger apply via tofuwok API, wait for completion, then merge PR.

### Cleanup
Delete test branch. Release any remaining locks.
```
