---
name: jira-poller
description: Continuously polls PSV Jira for new/updated P0/P1 tickets. Detects unprocessed tickets and spawns workflow tasks.
model: haiku
---

# Jira Poller Agent

## Purpose
Continuously monitor PSV Jira project for tickets that need processing. This is the entry point for the entire pipeline.

## Input
- Poll interval (default: 3 minutes)
- Priority filter (default: P0, P1)
- Project filter (default: PSV)

## Logic

### 1. Build JQL Query
```
project = PSV 
AND status in ("Open", "To Do", "In Progress", "Reopened") 
AND priority in ("Highest-P0", "High-P1") 
AND labels not in ("psv-im-processing", "psv-im-resolved", "psv-im-escalated")
ORDER BY priority ASC, created ASC
```

### 2. Execute Search
Use Atlassian MCP tool `searchJiraIssuesUsingJql` with fields:
- `summary, status, priority, components, assignee, reporter, created, updated, labels, customfield_11997, customfield_11800, customfield_11998, description, attachment`

### 3. Filter New Tickets
For each result:
1. Check workflow state DB/file — skip if ticket already in `processing` or `resolved` state
2. Check if ticket was updated since last poll (handle re-opened tickets)

### 4. For Each New Ticket
1. Add label `psv-im-processing` to the Jira ticket using `editJiraIssue`
2. Post initial comment:
   ```
   🤖 PSV-IM Agent — Processing Started
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   
   This ticket has been picked up for automated investigation.
   Pipeline: psv-im-agent | Priority: {priority}
   
   Next: Parsing ticket fields and fetching context...
   ```
3. Create workflow state file: `tasks/psv-im-agent/ticket-state-{KEY}.json`
4. Spawn Stage 1 (ticket-parser) as async task

### 5. Concurrency Guard
- Max concurrent workflows: 5 (configurable)
- If at capacity, queue ticket and log: "Ticket {KEY} queued — {N} workflows active"
- P0 tickets always preempt P1 queue

### 6. Health Check
Every 10 polls, log:
- Active workflows count
- Queued tickets count
- Last successful poll timestamp
- Jira API response time

## Output
- List of newly detected ticket keys
- Updated workflow state for each
- Spawned Stage 1 tasks

## Error Handling
- Jira API timeout → retry 3x with 5s backoff → log warning, continue next poll
- Jira 429 (rate limit) → wait 60s → retry
- Auth failure → log error, stop polling, alert
