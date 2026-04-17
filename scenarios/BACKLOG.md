# Scenario Backlog

Edge cases identified. Status from tofuwok developer review.

## State/Apply Timing

- [x] **Push after apply** — HANDLED (supersession). Written as `t2-lifecycle/push-after-apply`. Verify with test.
- [x] **Sequential applies (dir A then dir B)** — HANDLED (separate locks per dir, parallel). No scenario needed unless they share state.
- [ ] **Apply timeout / hangs forever** — PARTIAL. Temporal: 60min hard timeout. GHA: lock TTL 30min. No active watchdog. Lock janitor (F63) cleans up after. Worth testing GHA path.

## Merge Edge Cases

- [x] **Squash vs merge commit vs rebase** — HANDLED (agnostic, only checks `pr.GetMerged()` boolean). Worth a quick verification test.
- [x] **Merge with conflicts** — NOT OUR PROBLEM (GitHub blocks merge). No scenario needed.
- [x] **Fast-forward merge** — HANDLED (same Merged=true webhook). No scenario needed.

## Multi-Dir Interaction

- [x] **Shared module change fan-out** — HANDLED (`internal/affected/modules.go` ModuleGraph traces dependencies transitively). Worth testing to verify.
- [x] **Partial apply on multi-dir + close** — HANDLED (handlePRClosed force-releases remaining locks, dispatches applies for unapplied dirs if apply_mode=after_merge). Worth testing.

## Webhook/Event Edge Cases

- [ ] **Out-of-order events** — PARTIAL. cancelRunsForPR cancels in-flight plans, but no sequence tracking. If B arrives before A, no detection.
- [ ] **Duplicate webhook** — NOT HANDLED. X-GitHub-Delivery logged but never checked. Two identical webhooks create two runs + two dispatches. Lock contention prevents both executing but wasteful + confusing state. **File as tofuwok bug/feature.**

## Concurrent Edge Cases

- [x] **Two PRs same dir** — HANDLED (lock system). Already tested as `t3-multi-pr/lock-conflict`.
- [x] **Bot PR + human PR same dir** — HANDLED (locks are PR-scoped, bot PRs are just PRs). No special scenario needed.
- [ ] **Rapid PR creation (5 PRs in 30s)** — Untested. Likely handled but worth stress testing.

## Priority for Testing

### Should test (not handled or partial):
1. **Duplicate webhook** — create scenario, likely reveals a bug
2. **Apply timeout / GHA lock TTL expiry** — test heartbeat path
3. **Out-of-order synchronize** — hard to test but worth trying

### Should verify (handled but untested):
4. **Push after apply** — scenario written, needs to run
5. **Shared module fan-out** — write scenario
6. **Partial apply + close** — write scenario
7. **Squash merge** — quick verification

### Don't need scenarios:
- Merge with conflicts (GitHub problem)
- Fast-forward merge (same webhook)
- Sequential dir applies (terraform design, not ours)
- Bot vs human PRs (just PRs)

## Real Infra (T3)

- [ ] **Drift detection** — at `t3-real-infra/drift-detection`
- [ ] **Provider auth failure** — at `t3-real-infra/provider-auth-failure`
- [ ] **Actual S3 backend** — plan/apply with real S3 state
- [ ] **Cross-account assume role** — different AWS accounts per company dir
- [ ] **Cost estimation** — infracost on plan output
