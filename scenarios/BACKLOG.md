# Scenario Backlog

Edge cases and scenarios identified but not yet written. Prioritize as needed.

## State/Apply Timing

- [ ] **Sequential applies on same PR** — apply dir A, then apply dir B. Does B's plan see A's state changes?
- [ ] **Apply timeout** — terraform apply hangs forever. Lock held indefinitely? Heartbeat should catch it.
- [ ] **Push after apply** — ✅ written as `t2-lifecycle/push-after-apply`

## Merge Edge Cases

- [ ] **Squash merge vs merge commit vs rebase** — does tofuwok handle all three merge strategies? Lock release, affected detection.
- [ ] **Merge with conflicts** — PR has merge conflicts with main. What happens to locks? Can you still apply?
- [ ] **Fast-forward merge** — no merge commit created. Does affected detection still work on push to main?

## Multi-Dir Interaction

- [ ] **Shared module change fan-out** — change `test/modules/label/`, verify tofuwok plans all consuming dirs (alpha/snowflake, bravo/snowflake, charlie/snowflake)
- [ ] **Partial apply on multi-dir** — PR touches 6 dirs, apply only 3, close PR. Orphaned applies on 3 dirs, unapplied locks on 3.

## Webhook/Event Edge Cases

- [ ] **Duplicate webhook** — same PR event delivered twice. Plan should be idempotent, not dispatch twice.
- [ ] **Out-of-order events** — synchronize arrives before previous plan finishes. Supersession should handle it.
- [ ] **Webhook delivery failure + retry** — GitHub retries after 10s. Tofuwok should handle the delayed duplicate.

## Concurrent Edge Cases

- [ ] **Same user, two PRs, same dir** — both plan, lock conflict. Normal flow but worth verifying the UX.
- [ ] **Bot PR + human PR same dir** — Renovate bumps a provider, human changes resources. Lock contention between bot and human.
- [ ] **Rapid PR creation** — 5 PRs opened in 30 seconds, all touching different dirs. Tofuwok should handle the burst.

## Error Recovery

- [ ] **Backend down during plan** — S3 backend unreachable. Plan fails on init. Error surfaced correctly?
- [ ] **State lock contention** — terraform state lock held by another process. Plan fails with lock error. Tofuwok should surface this differently from a code error.
- [ ] **OOM on large plan** — terraform runs out of memory on a huge plan. GHA runner killed. Heartbeat detects orphan.

## Real Infra (T3)

- [ ] **Drift detection** — moved to `t3-real-infra/drift-detection`
- [ ] **Provider auth failure** — moved to `t3-real-infra/provider-auth-failure`
- [ ] **Actual S3 backend** — plan/apply with real S3 state, not local backend
- [ ] **Cross-account assume role** — different AWS accounts per company dir
- [ ] **Cost estimation** — infracost on plan output, verify cost is recorded
