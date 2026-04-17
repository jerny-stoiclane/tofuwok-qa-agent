# BUG-001: Lock `applied` flag not set to true after successful apply

**Severity:** medium
**Component:** tofuwok-api
**Found in:** t2-single-pr/multi-dir-plan
**Run ID:** 20260417-144041
**Date:** 2026-04-17

## Summary
After a successful apply run completes, the lock's `applied` field remains `false`. It should be set to `true` to indicate the apply succeeded before merge.

## Reproduce
- GIVEN a PR with changes in one or more dirs, plan succeeded, lock acquired with `applied=false`
- WHEN apply is triggered via `bin/twk trigger --pr 15 --action apply` and completes with `status=success`
- THEN `bin/twk locks` shows `applied=false` for all dirs

## Expected
- THEN `bin/twk locks` should show `applied=true` for all dirs where apply succeeded

## Evidence
Checked locks BEFORE merging (so lock release didn't interfere):

```text
$ bin/twk locks
[
  {"dir": "test/companies/charlie/snowflake",           "pr_number": 15, "applied": false},
  {"dir": "test/companies/bravo/snowflake",             "pr_number": 15, "applied": false},
  {"dir": "test/companies/bravo/aws/us-east-1/platform","pr_number": 15, "applied": false},
  {"dir": "test/companies/alpha/snowflake",             "pr_number": 15, "applied": false},
  {"dir": "test/companies/alpha/aws/us-east-1/platform","pr_number": 15, "applied": false},
  {"dir": "test/companies/_global/networking",          "pr_number": 15, "applied": false}
]
```

Apply runs all completed successfully:
```text
$ bin/twk runs --pr 15 --type apply
All 6 runs: status=success
```

Apply check runs all posted:
```text
$ gh api commits/{sha}/check-runs | jq apply
tofuwok/apply = success
tofuwok/apply/{dir} = success (all 6)
```

## Context
- PR: #15
- SHA: d11799096563669b4c393998e21912d19b638590
- Results file: results/multi-dir-plan-20260417-144041.md
- Also observed in t2-single-pr/plan-apply-merge (PR #14) but was inconclusive there (checked after merge)
