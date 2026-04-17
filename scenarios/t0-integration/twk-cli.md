---
name: twk-cli
tier: t0
timeout: 30
---

# T0: bin/twk CLI Verification

Verify every bin/twk subcommand works without errors and returns parseable output. This catches broken jq filters, null handling issues, and API response format changes.

## Phases

### Phase 1: Help
```bash
bin/twk help
```
Assert: prints usage, exit 0

### Phase 2: Health
```bash
bin/twk health
```
Assert: prints "ok" or "DOWN", exit 0

### Phase 3: Repos
```bash
bin/twk repos | jq 'length'
```
Assert: returns number, no jq errors

### Phase 4: Runs — no filter
```bash
bin/twk runs --pr 99999 | jq 'length'
```
Assert: returns 0, no errors

### Phase 5: Runs — with filter
```bash
bin/twk runs --pr 9 --type plan | jq 'length'
```
Assert: returns number >= 0, no errors

### Phase 6: Locks — empty
```bash
bin/twk locks | jq 'length'
```
Assert: returns number >= 0, no jq errors

### Phase 7: Locks — specific dir
```bash
bin/twk locks --dir nonexistent/dir
```
Assert: returns `{}` or empty, no errors

### Phase 8: Wait-plans — immediate timeout
```bash
bin/twk wait-plans --pr 99999 --dirs fake/dir --timeout 5 2>&1
```
Assert: returns TIMEOUT, exit 1

### Phase 9: Cleanup — noop
```bash
bin/twk cleanup
```
Assert: runs without error, reports "CLEANUP DONE"
