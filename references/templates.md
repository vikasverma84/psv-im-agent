# Jira Comment Templates

Templates used by the PSV-IM Agent for posting structured comments to Jira tickets.

## Template 1: Processing Started (Stage 0)

```
🤖 PSV-IM Agent — Processing Started
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This ticket has been picked up for automated investigation.

📋 Ticket: {{ticket_id}}
⚡ Priority: {{priority}}
🏢 Brand: {{brand}}
🔧 Component: {{component}}

Pipeline: psv-im-agent v1.0
Next: Parsing ticket fields and fetching context...
```

## Template 2: Parsing Complete (Stage 1)

```
🤖 PSV-IM Agent — Ticket Parsed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Fields extracted:
• Brand: {{brand}}
• Component: {{component}} → Team: {{team_skill}}
• Environment: {{environment}}
• Geo: {{geo_region}}
• Org ID: {{org_id}}
{{#if dataflow_id}}• Dataflow: {{dataflow_id}}{{/if}}
{{#if customer_id}}• Customer: {{customer_id}}{{/if}}
{{#if api_endpoint}}• API: {{api_endpoint}}{{/if}}

Next: Fetching BRD/SDD context + running RCA...
```

## Template 3: RCA Complete (Stage 2b)

```
🤖 PSV-IM Agent — Root Cause Analysis
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

─── 5-Whys Analysis ───

{{#each why_chain}}
Why #{{level}}: {{question}}
→ {{answer}}
  Evidence needed: {{evidence_needed}}

{{/each}}

─── Hypothesis ───
{{root_cause_hypothesis}}

Confidence: {{confidence}}%

Next: Routing to {{team_skill}} for investigation...
```

## Template 4: KB Fast Path (Stage 3)

```
🤖 PSV-IM Agent — Known Issue Detected ⚡
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Similar to: {{past_ticket_key}} (resolved {{resolved_date}})
Match confidence: {{similarity}}%

Past root cause: {{past_root_cause}}
Past resolution: {{past_resolution}}

Applying known fix. Skipping to approval gate...
```

## Template 5: Routed to Team Skill (Stage 4)

```
🤖 PSV-IM Agent — Routed to Team Skill
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Component: {{component}}
Team Skill: {{skill_id}}
Status: Investigation in progress...

RCA Hypothesis: {{root_cause_hypothesis}}
Data being fetched:
{{#each data_requests}}
  • {{api}}
{{/each}}
```

## Template 6: Escalation (Any Stage)

```
🤖 PSV-IM Agent — Escalation Required ⚠️
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Reason: {{escalation_reason}}

{{#if partial_rca}}
─── Partial Findings ───
{{partial_rca}}
{{/if}}

{{#if no_skill}}
No team skill registered for component "{{component}}".
{{/if}}

{{#if qa_failed}}
QA failed after {{retry_count}} retries.
Last QA score: {{qa_score}}/13
{{/if}}

Action required: Manual investigation needed.
Please assign to appropriate team member.
```

## Template 7: QA Review (Stage 6)

```
🤖 PSV-IM Agent — QA Review
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Verdict: {{verdict}}
Score: {{total_score}}/13

Investigation: {{investigation_score}}/4 — {{investigation_status}}
Fix Proposal:  {{fix_score}}/5 — {{fix_status}}
Test Results:  {{test_score}}/4 — {{test_status}}

{{#if warnings}}
⚠️ Warnings:
{{#each warnings}}
  • {{this}}
{{/each}}
{{/if}}

{{#if proceed}}
Next: Posting for human approval...
{{else}}
Retrying investigation (attempt {{retry_count}}/3)...
{{/if}}
```

## Template 8: Approval Request (Stage 7)

```
🤖 PSV-IM Agent — Approval Required
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Ticket: {{ticket_id}} | Priority: {{priority}} | Brand: {{brand}}
🔧 Component: {{component}} | Team Skill: {{skill_id}}

─── Root Cause Analysis ───
{{#each why_chain}}
{{level}}. {{question}} → {{answer}}
{{/each}}

Confirmed root cause: {{root_cause}}
Confidence: {{confidence}}%

─── Evidence ───
{{#each evidence}}
• [{{source}}] {{data_summary}}
{{/each}}

─── Proposed Fix ───
Type: {{fix_type}}
Environment: {{fix_environment}} ⚠️
Description: {{fix_description}}
Reversible: {{reversible}}
Risk: {{risk_level}}

{{#each changes}}
  File: {{file}}
  Change: {{before}} → {{after}}
{{/each}}

─── Test Results ───
Tests: {{tests_run}} run | {{tests_passed}} passed | {{tests_failed}} failed
Smoke: {{smoke_status}}
{{#each test_details}}
  • {{name}}: {{status}} ({{duration_ms}}ms)
{{/each}}

─── QA Verdict ───
Score: {{qa_score}}/13 | {{qa_verdict}}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ACTION REQUIRED — Reply with one of:
  ✅ APPROVE — apply fix to {{fix_environment}}
  ❌ REJECT — with reason
  ❓ MORE INFO — specify what's needed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Template 9: Resolution Complete (Stage 8)

```
🤖 PSV-IM Agent — Resolution Complete ✅
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 {{ticket_id}} | {{priority}} | {{brand}} | {{component}}

─── Summary ───
{{resolution_summary}}

─── Root Cause ───
{{root_cause}}

─── Fix Applied ───
{{fix_description}}
Environment: {{fix_environment}}
Applied at: {{fix_timestamp}}

─── Timeline ───
• Detected:     {{detected_at}}
• Parsed:       {{parsed_at}}
• RCA Complete: {{rca_at}}
• Fix Proposed: {{fix_proposed_at}}
• Approved by:  {{approver}} at {{approved_at}}
• Resolved:     {{resolved_at}}
• Total Time:   {{total_duration}}

─── Metrics ───
• First Response: {{first_response_time}}
• Resolution: {{resolution_time}}
• Auto-resolved: {{auto_resolved}}
• KB Match: {{kb_used}}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pipeline: psv-im-agent v1.0 | Skill: {{skill_id}}
```
