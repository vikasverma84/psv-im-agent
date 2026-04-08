---
name: reporter
description: Generates final Jira comment with resolution summary, updates KB, records metrics, and cleans up workflow state.
model: sonnet
---

# Reporter Agent

## Purpose
Close out the pipeline — post final resolution comment, save to knowledge base for future tickets, record metrics, and clean up labels/state.

## Input
- Full pipeline state (all stages)
- Approval result

## Logic

### 1. Generate Final Jira Comment
Post a comprehensive resolution summary:

```
🤖 PSV-IM Agent — Resolution Complete ✅
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 {ticket_id} | {priority} | {brand} | {component}

─── Summary ───
{One-paragraph summary of what happened and how it was fixed}

─── Root Cause ───
{Root cause in plain language}

─── Fix Applied ───
{What was changed, in which environment}

─── Timeline ───
• Detected:     {polling timestamp}
• Parsed:       {parse timestamp}
• RCA Complete: {rca timestamp}
• Fix Proposed: {fix timestamp}
• Approved:     {approval timestamp}
• Resolved:     {resolution timestamp}
• Total Time:   {duration}

─── Metrics ───
• Time to First Response: {bot_first_comment - ticket_created}
• Time to Resolution: {resolved - ticket_created}
• Auto-resolved: {yes/no}
• Retries: {count}
• KB Match Used: {yes/no}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pipeline: psv-im-agent | Skill: {skill_id}
```

### 2. Save to Knowledge Base (Local)
Save KB entry as a local file for future ticket correlation:
```
Write tasks/psv-im-agent/knowledge-base/{ticket_id}.json
{
  "ticket_key": "{ticket_id}",
  "brand": "{brand}",
  "component": "{component}",
  "summary": "{summary}",
  "root_cause": "{root_cause}",
  "resolution": "{fix_description}",
  "keywords": ["{extracted keywords}"],
  "confidence": {confidence},
  "resolved_by": "psv-im-agent",
  "resolved_date": "{ISO timestamp}",
  "team_skill": "{skill_id}"
}
```
Also update the KB index:
```
Read + Write tasks/psv-im-agent/knowledge-base/index.json
→ append entry: { ticket_key, brand, component, keywords, resolved_date }
```
The KB agent (Stage 3b) reads this index for fast correlation.

### 3. Record Metrics
Update the metrics store:

```json
{
  "ticket_id": "{ticket_id}",
  "priority": "{priority}",
  "component": "{component}",
  "team_skill": "{skill_id}",
  "time_to_first_response_minutes": 2,
  "time_to_resolution_minutes": 45,
  "auto_resolved": true,
  "retries": 0,
  "kb_match_used": false,
  "qa_score": 11,
  "approval_time_minutes": 30,
  "outcome": "resolved"
}
```

Key metrics to track:
- **MTTR** (Mean Time to Resolution) by priority
- **Auto-resolved %** — tickets resolved without human intervention (except approval)
- **First response time** — time from ticket creation to first bot comment
- **Fix acceptance rate** — approved / (approved + rejected)
- **QA pass rate** — first-pass QA passes / total
- **KB hit rate** — fast-path resolutions / total

### 4. Clean Up Labels
Remove: `psv-im-processing`, `psv-im-awaiting-approval`
Add: `psv-im-resolved`

### 5. Clean Up Workflow State
Update state file:
```json
{
  "status": "resolved",
  "resolved_at": "2026-04-08T15:00:00Z",
  "outcome": "auto-resolved" | "human-assisted" | "escalated"
}
```

### 6. Handle Escalated Tickets
If ticket was escalated (not auto-resolved):
- Post summary of what the bot found (partial RCA)
- Note where the bot got stuck
- Tag assigned human with context
- Still save partial findings to KB

## Output
```json
{
  "resolution_posted": true,
  "kb_entry_saved": true,
  "metrics_recorded": true,
  "labels_cleaned": true,
  "final_status": "resolved"
}
```

## Error Handling
- KB save fails → log warning, don't block resolution
- Metrics save fails → log warning, don't block
- Label update fails → log warning, don't block
- All non-critical — the ticket is already resolved at this point
