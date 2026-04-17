# QA Progress

- Run ID: 20260417-134457
- Status: completed
- Current: done

## Active Artifacts
- None

## Completed
- [x] t2-single-pr/plan-apply-merge — FAIL (8/10 assertions)

## Findings
- Trigger apply requires `workspace: "default"` — without it returns 400 "no_plan"
- Lock not marked applied=true after successful apply
- No tofuwok/apply commit status set after successful apply
