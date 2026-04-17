---
name: endpoints
tier: t0
timeout: 60
---

# T0: API Endpoint Verification

Hit every tofuwok API endpoint the QA agent uses. Verify response shape, status codes, and that `bin/twk` parses them correctly. No PRs, no terraform — pure API testing.

## Phases

### Phase 1: Health
```bash
bin/twk health
```
Assert: returns "ok"

### Phase 2: Repos
```bash
bin/twk repos
```
Assert: returns JSON array, contains terraform-orchestrator-gha with execution_mode=gha

### Phase 3: Runs (empty filter)
```bash
bin/twk runs --pr 99999
```
Assert: returns empty array `[]`, no errors

### Phase 4: Runs (real data)
```bash
bin/twk runs --pr 9 --type plan
```
Assert: returns array of run objects, each has: id, status, dir, run_type, has_changes

### Phase 5: Locks (list all)
```bash
bin/twk locks
```
Assert: returns array (may be empty), no jq errors

### Phase 6: Locks (check specific dir)
```bash
bin/twk locks --dir test/companies/bravo/snowflake
```
Assert: returns object or empty `{}`, no errors

### Phase 7: Stacks
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/stacks" | tr -d '\000-\037' | jq '.'
```
Assert: returns JSON with data array, each entry has owner, repo, dir, workspace, last_run_status

### Phase 8: PRs
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/prs" | tr -d '\000-\037' | jq '.'
```
Assert: returns JSON with data array

### Phase 9: Events
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/events" | tr -d '\000-\037' | jq '. | length'
```
Assert: returns number (may be 0), no errors

### Phase 10: Effective Config
```bash
curl -sf -H "Authorization: Bearer $TOFUWOK_TOKEN" "$TOFUWOK_API/api/v1/repos/jerny-stoiclane/terraform-orchestrator-gha/config/effective" | tr -d '\000-\037' | jq '.'
```
Assert: returns JSON with merged config object

### Phase 11: Trigger (dry validation — bad PR)
```bash
bin/twk trigger --pr 0 --action plan
```
Assert: returns error (400 or similar), does NOT dispatch a workflow

### Phase 12: Swagger Doc
```bash
curl -sf http://localhost:8080/swagger/doc.json | jq '.paths | keys | length'
```
Assert: returns number > 10 (endpoints exist)
