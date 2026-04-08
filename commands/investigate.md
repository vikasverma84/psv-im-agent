---
description: Investigate a specific PSV ticket — runs the full pipeline (parse → context → RCA → route → QA → approve) on a single ticket.
argument-hint: <ticket-key> [--resume] [--skip-approval]
---

# /investigate — Single Ticket Investigation

## Usage
```
/investigate PSV-12345
/investigate PSV-12345 --resume          # Resume from last stage
/investigate PSV-12345 --skip-approval   # Skip approval gate (DEMO only)
```

## Logic

### 1. Parse Arguments
```bash
TICKET_KEY="$1"
RESUME=false
SKIP_APPROVAL=false

for arg in "$@"; do
  case "$arg" in
    --resume) RESUME=true ;;
    --skip-approval) SKIP_APPROVAL=true ;;
  esac
done
```

### 2. Validate Ticket Key
- Must match pattern: `PSV-\d+`
- Fetch ticket from Jira to confirm it exists
- If not found → error: "Ticket {key} not found in Jira"

### 3. Check for Resume
If `--resume`:
1. Load state file: `tasks/psv-im-agent/ticket-state-{KEY}.json`
2. If exists, resume from `current_stage`
3. If not exists → start fresh

### 4. Run Pipeline
Execute stages sequentially:

```
Stage 1: ticket-parser
  ↓ save parsed ticket to state
Stage 2a + 2b: context-agent + five-whys-agent (parallel)
  ↓ save context + RCA to state
Stage 3: kb-agent
  ↓ if fast-path, skip to Stage 7
Stage 4: component-router
  ↓ dispatch to team skill
Stage 5: (team skill runs)
  ↓ receive output
Stage 6: qa-agent
  ↓ if fail, retry Stage 5 (max 3x)
Stage 7: approval-gate (skip if --skip-approval)
  ↓ wait for human
Stage 8: reporter
  ↓ done
```

### 5. State Persistence
After each stage, save state to:
`tasks/psv-im-agent/ticket-state-{KEY}.json`

This enables `--resume` to pick up from the last completed stage.

### 6. Progress Output
Print progress to console:
```
[PSV-12345] Stage 1: Parsing ticket fields...
[PSV-12345] Stage 1: ✓ Brand: Abbott_HK_Prod | Component: Integrations | Priority: P0
[PSV-12345] Stage 2: Fetching BRD/SDD context + running 5-Whys RCA...
[PSV-12345] Stage 2: ✓ SDD found (3 relevant sections) | RCA confidence: 65%
[PSV-12345] Stage 3: Checking knowledge base...
[PSV-12345] Stage 3: ✓ No high-confidence match. Proceeding to full investigation.
[PSV-12345] Stage 4: Routing to integration-skill...
[PSV-12345] Stage 5: Integration skill investigating...
[PSV-12345] Stage 5: ✓ Root cause confirmed. Fix proposed.
[PSV-12345] Stage 6: QA validation...
[PSV-12345] Stage 6: ✓ QA PASSED (11/13)
[PSV-12345] Stage 7: Awaiting human approval on Jira...
[PSV-12345] Stage 7: ✓ Approved by sa.lead@capillarytech.com
[PSV-12345] Stage 8: Posting resolution + updating KB...
[PSV-12345] ✅ RESOLVED — Total time: 47 minutes
```

### 7. Error Handling
- Stage failure → save state, post error comment on Jira, exit with error
- Use `--resume` to retry from the failed stage after fixing the issue
- Network timeouts → retry 3x before failing
