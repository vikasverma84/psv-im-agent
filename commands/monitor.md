---
description: Start the continuous Jira polling loop. Monitors PSV project for new P0/P1 tickets and auto-dispatches them through the pipeline.
argument-hint: [--interval <minutes>] [--priorities <P0,P1>] [--dry-run]
---

# /monitor — Continuous Jira Polling

## Usage
```
/monitor                              # Default: 3 min interval, P0+P1
/monitor --interval 5                 # Poll every 5 minutes
/monitor --priorities P0,P1,P2        # Include P2 tickets
/monitor --dry-run                    # Show what would be picked up, don't process
```

## Logic

### 1. Parse Arguments
```bash
INTERVAL=3        # minutes
PRIORITIES="P0,P1"
DRY_RUN=false

for arg in "$@"; do
  case "$arg" in
    --interval) shift; INTERVAL="$1" ;;
    --priorities) shift; PRIORITIES="$1" ;;
    --dry-run) DRY_RUN=true ;;
  esac
done
```

### 2. Start Polling Loop
```
🤖 PSV-IM Agent — Monitor Started
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Interval: every {INTERVAL} minutes
Priorities: {PRIORITIES}
Projects: PSV
Mode: {LIVE / DRY-RUN}
Max concurrent: 5
```

### 3. Each Poll Cycle
1. Invoke `jira-poller` agent with current config
2. For each new ticket detected:
   - If DRY_RUN: print ticket details, don't process
   - If LIVE: spawn `/investigate {KEY}` as async task
3. Print status:
```
[Poll #17 @ 10:45:00] Found 2 new tickets: PSV-12345 (P0), PSV-12346 (P1)
[Poll #17] Active workflows: 3/5 | Queued: 0
```

### 4. Dashboard Summary (every 10 polls)
```
═══ PSV-IM Agent Dashboard ═══
Active:   PSV-12345 (Stage 5 — integration-skill)
          PSV-12346 (Stage 2 — fetching context)
Queued:   0
Resolved: PSV-12300, PSV-12298
Escalated: PSV-12290 (no webapp-skill registered)
═══════════════════════════════
```

### 5. Graceful Shutdown
- Ctrl+C or SIGTERM → finish active stages, save all state, exit cleanly
- Resume with `/monitor` — picks up where it left off via state files

### 6. Error Handling
- Jira API down → log warning, retry next cycle
- All 5 slots full → queue new tickets, process FIFO when slot opens
- P0 preemption: if at capacity and P0 arrives, pause lowest-priority P1 workflow
