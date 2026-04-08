---
name: psv-im-agent
description: >
  PSV-IM Agent — Self-resolving incident management hub for PSV delivery.
  Autonomous Jira polling, ticket parsing, component-based routing to team skills,
  root-cause analysis, fix generation, QA validation, and human approval gate.
  Triggers on: PSV ticket, incident, bug triage, auto-resolve, monitor PSV,
  investigate ticket, production issue, P0, P1, dataflow error, integration bug.
  Scope: PSV delivery — CRs, incidents, BAU across Integration, WebApp, MobileApp, Config teams.
compatibility:
  tools:
    - mcp__claude_ai_Capillary_Docs__search-endpoints
    - mcp__claude_ai_Capillary_Docs__get-endpoint
    - mcp__claude_ai_Capillary_Docs__execute-request
    - mcp__claude_ai_Atlassian__searchJiraIssuesUsingJql
    - mcp__claude_ai_Atlassian__getJiraIssue
    - mcp__claude_ai_Atlassian__editJiraIssue
    - mcp__claude_ai_Atlassian__addCommentToJiraIssue
    - mcp__claude_ai_Atlassian__getConfluencePage
    - mcp__claude_ai_Atlassian__searchConfluenceUsingCql
  skills: []
model: opus
---

# PSV-IM Agent — Hub Orchestrator

## Overview

The PSV-IM Agent is an autonomous incident management hub that monitors PSV Jira for client-specific bugs, investigates root causes using Capillary platform APIs, and routes tickets to specialized team skills based on the PSV component field. It does not resolve tickets itself — it orchestrates the pipeline and delegates domain-specific investigation and fixes to pluggable team skills.

```
                    ┌──────────────────────┐
                    │   PSV-IM Agent Hub    │
                    │                      │
                    │  Poll → Parse → Route│
                    │  → Approve → Report  │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
     │ Integration   │ │ WebApp       │ │ Mobile       │ ...
     │ Skill (slot)  │ │ Skill (slot) │ │ Skill (slot) │
     └───────────────┘ └──────────────┘ └──────────────┘
       Team builds &     Team builds &    Team builds &
       attaches own      attaches own     attaches own
```

## Core Principles

1. **Hub, not monolith** — The agent owns the pipeline (poll → parse → context → route → approve → report). Domain-specific investigation and fixes are delegated to team skills.
2. **Component is the routing key** — The PSV Jira `components` field determines which team skill handles investigation and resolution.
3. **Pluggable team skills** — Each team registers their skill with a manifest. The hub discovers and invokes them. No team skill = escalate to human.
4. **Human approval required** — The agent never deploys to production autonomously. All fixes require human approval at the gate.
5. **Concurrent processing** — Multiple P0/P1 tickets processed in parallel, each in an isolated workflow.
6. **Listening-first** — Continuous Jira polling (no webhook dependency). Webhooks are a bonus, not a requirement.

## Pipeline — 9 Stages

```
Stage 0: Jira Poller (continuous loop)
    │
    ▼
Stage 1: Ticket Parser (extract fields)
    │
    ▼
Stage 2a: BRD/SDD Context Agent (fetch docs)
    │
    ▼
Stage 2b: Classifier Agent (validate concern against BRD/SDD)
    │
    ├── Bug → continue pipeline
    ├── SDD Gap → escalate to SA, STOP
    ├── BRD Gap → escalate to PM, STOP
    └── Change Request → route to CR workflow, STOP
    │
    ▼ (Bug only)
Stage 3a: 5-Whys RCA Agent     (parallel)     Stage 3b: KB Correlation
    │                                               │
    └──────────────┬────────────────────────────────┘
                   ▼
Stage 4: Component Router → dispatches to team skill
    │
    ▼
Stage 5: Team Skill executes (investigate → fix → test)
    │
    ▼
Stage 6: QA Agent (validates team skill output)
    │
    ├── QA Passed → Stage 7
    └── QA Failed → retry Stage 5 (max 3x) → escalate
    │
    ▼
Stage 7: Approval Gate (human reviews RCA + fix + test results)
    │
    ├── Approved → deploy + close ticket
    └── Rejected → learning + re-investigate or escalate
    │
    ▼
Stage 8: Reporter (final Jira comment + KB update + metrics)
```

## Stage Details

### Stage 0: Jira Poller

**Agent**: `jira-poller`
**Trigger**: Continuous loop every N minutes (configurable, default 3 min)
**Scope**: PSV project, P0/P1 tickets (expandable to all priorities)

Logic:
1. Query Jira: `project = PSV AND status in ("Open", "To Do", "In Progress") AND priority in ("Highest-P0", "High-P1") AND labels not in ("psv-im-processing", "psv-im-resolved")`
2. For each new ticket not already in the workflow state DB:
   - Add label `psv-im-processing` to Jira ticket
   - Create workflow state record: `{ ticket_id, status: "polling", stage: 0 }`
   - Spawn async task for Stage 1
3. Concurrency: each ticket gets its own isolated workflow (asyncio.Task)
4. Dedup: skip tickets already in `processing` or `resolved` state

### Stage 1: Ticket Parser

**Agent**: `ticket-parser`
**Input**: Jira ticket key (e.g., PSV-12345)
**Output**: Parsed ticket object

Extract from Jira issue fields:
```json
{
  "ticket_id": "PSV-12345",
  "summary": "...",
  "description": "...",
  "brand": "Abbott_HK_Prod",           // customfield_11997
  "component": "Integrations",          // components[0].name
  "environment": "Production",          // customfield_11800
  "geo_region": "APAC",                 // customfield_11998
  "priority": "P0",                     // priority.name
  "reporter": "john.doe@capillary...",
  "assignee": null,
  "org_id": "1115",                     // extracted from description or brand config
  "dataflow_id": null,                  // extracted from description if present
  "customer_id": null,                  // extracted from description if present
  "api_endpoint": null,                 // extracted from description if present
  "attachments": [],
  "created": "2026-04-08T10:00:00Z"
}
```

Field extraction rules:
- `brand`: from `customfield_11997[0].value`
- `component`: from `components[0].name` — **this is the routing key**
- `org_id`: regex from description (`org[_\s]?id[:\s]*(\d+)`) or lookup from brand config
- `dataflow_id`: regex from description (`dataflow[_\s]?(?:id|uuid)[:\s]*([a-f0-9-]+)`)
- `customer_id`: regex from description (`customer[_\s]?id[:\s]*(\d+)`)
- `api_endpoint`: regex from description (`(?:GET|POST|PUT|DELETE)\s+(/[^\s]+)`)

### Stage 2a: BRD/SDD Context Agent

**Agent**: `context-agent`
**Input**: brand, component, summary
**Output**: SDD + BRD content relevant to the concern

Logic:
1. Check chatbot DB (`brand_documents` table) for saved SDD/BRD for this brand
2. If not found, search Confluence (SA + AG spaces) using brand aliases
3. Extract relevant sections based on component and concern keywords
4. Return structured context: `{ sdd_content, brd_content, relevant_sections[] }`

### Stage 2b: Classifier Agent (BRD/SDD Validation)

**Agent**: `classifier-agent`
**Input**: parsed ticket + BRD/SDD context from Stage 2a
**Output**: Classification + pipeline action

**This is the critical gate** — validates the reported concern against design documents before running RCA.

Classification matrix:

| SDD Coverage | BRD Coverage | Classification | Pipeline Action |
|---|---|---|---|
| COVERED | COVERED | **Bug** | `INVESTIGATE` → continue to Stage 3 |
| NOT COVERED | COVERED | **SDD Gap** | `ESCALATE_SA` → assign to SA, STOP |
| NOT COVERED | NOT COVERED | **BRD Gap** | `ESCALATE_PM` → assign to PM, STOP |
| (any) | (any) + enhancement language | **Change Request** | `ROUTE_CR` → CR workflow, STOP |

Only **Bug** classification proceeds to Stage 3. All others post a structured Jira comment explaining the classification with relevant BRD/SDD sections cited, then stop the pipeline.

If no BRD/SDD available → defaults to Bug with low confidence (0.3) and proceeds.

### Stage 3a: 5-Whys RCA Agent

**Agent**: `five-whys-agent`
**Input**: parsed ticket, BRD/SDD context
**Output**: Root cause chain + hypothesis

Logic:
1. Start with the reported symptom (ticket summary + description)
2. Ask "Why?" iteratively (up to 5 levels), using:
   - SDD/BRD context (expected behavior)
   - Ticket description (actual behavior)
   - Similar past tickets from KB
3. At each level, generate a hypothesis and identify what data would confirm/deny it
4. Output:
```json
{
  "symptom": "Customer points not crediting after transaction",
  "why_chain": [
    { "level": 1, "why": "Points calculation returned 0", "evidence_needed": "evaluation log" },
    { "level": 2, "why": "Promotion rule not matching", "evidence_needed": "rule config" },
    { "level": 3, "why": "Customer tier not updated", "evidence_needed": "status log" },
    { "level": 4, "why": "Tier update dataflow failed", "evidence_needed": "debug-mode/error-events" },
    { "level": 5, "why": "SFTP source file missing", "evidence_needed": "debug-mode/jobs" }
  ],
  "root_cause_hypothesis": "SFTP file delivery failure causing tier dataflow to skip, leaving customer at wrong tier, causing promotion rule mismatch",
  "confidence": 0.65,
  "data_requests": ["debug-mode/error-events", "debug-mode/jobs", "evaluation log"]
}
```

### Stage 3: KB Correlation

**Agent**: `kb-agent`
**Input**: parsed ticket, 5-whys output
**Output**: KB match result

Logic:
1. Search knowledge base for similar issues (brand + component + keywords)
2. If match confidence > 80%:
   - Skip to known fix (fast path)
   - Post Jira comment: "Similar to {past_ticket}. Known resolution: {fix}. Applying..."
3. If match confidence 50-80%:
   - Include as reference, but proceed with full investigation
4. If no match:
   - Proceed normally

### Stage 4: Component Router

**Agent**: `component-router`
**Input**: parsed ticket (with component field), context, 5-whys output
**Output**: Routed to appropriate team skill

Component → Team Skill mapping (from `references/component-map.md`):

| Component | Team Skill | Skill ID |
|---|---|---|
| Integrations | Integration Skill | `integration-skill` |
| Data-Ingestion | Integration Skill | `integration-skill` |
| Webapp-CRM | WebApp Skill | `webapp-skill` |
| MobileApp-CRM | Mobile App Skill | `mobile-skill` |
| Configurations | Config Skill | `config-skill` |
| US_Configuration | Config Skill | `config-skill` |
| BAU-QA | QA Skill | `qa-skill` |
| QA_task | QA Skill | `qa-skill` |
| Solution-Consulting | Escalate to SA | `ESCALATE` |
| SI-Partner | Escalate to Partner | `ESCALATE` |

Router logic:
1. Read component from parsed ticket
2. Look up team skill ID in component map
3. Check if team skill is registered (manifest exists at `~/.claude/skills/{skill-id}/`)
4. If registered:
   - Pass standardized input to team skill (see Skill Interface Contract in references)
   - Team skill returns: `{ investigation, fix_proposal, test_results }`
5. If NOT registered:
   - Post Jira comment: "No team skill registered for component '{component}'. Escalating to human."
   - Add label `psv-im-needs-human`
   - Stop pipeline for this ticket

### Stage 5: Team Skill Execution

**Not owned by the hub** — this is where the team skill runs.

The hub passes a standardized input payload:
```json
{
  "ticket": { /* parsed ticket from Stage 1 */ },
  "context": { /* BRD/SDD from Stage 2a */ },
  "rca": { /* 5-Whys output from Stage 2b */ },
  "kb_matches": [ /* from Stage 3 */ ],
  "brand_credentials": { /* cluster, till_id, org_id */ },
  "data_requests": [ /* what data the RCA needs */ ]
}
```

The team skill returns a standardized output:
```json
{
  "skill_id": "integration-skill",
  "investigation": {
    "findings": "...",
    "evidence": [ { "source": "debug-mode/error-events", "data": "..." } ],
    "root_cause_confirmed": true,
    "root_cause": "SFTP connection timeout causing dataflow failure"
  },
  "fix_proposal": {
    "type": "dataflow_config",
    "description": "Increase SFTP retry count from 1 to 3, add timeout of 60s",
    "changes": [ { "file": "dataflow.json", "diff": "..." } ],
    "environment": "DEMO",
    "reversible": true
  },
  "test_results": {
    "smoke_passed": true,
    "tests_run": 5,
    "tests_passed": 5,
    "details": "..."
  }
}
```

### Stage 6: QA Agent

**Agent**: `qa-agent`
**Input**: team skill output (investigation + fix + test results)
**Output**: QA verdict (pass/fail)

Logic:
1. Validate investigation completeness:
   - Root cause identified? Evidence attached?
   - Does root cause align with 5-Whys hypothesis?
2. Validate fix proposal:
   - Is fix scoped to DEMO only?
   - Is fix reversible?
   - Does fix address root cause (not just symptom)?
3. Validate test results:
   - All smoke tests passed?
   - No regression detected?
4. If QA PASSES → proceed to Stage 7
5. If QA FAILS:
   - Increment retry counter
   - If retries < 3 → re-run Stage 5 with QA feedback
   - If retries >= 3 → escalate to human, add label `psv-im-needs-human`

### Stage 7: Approval Gate

**Agent**: `approval-gate`
**Input**: Full pipeline output (ticket + RCA + fix + test results)
**Output**: Approval decision

Logic:
1. Post comprehensive Jira comment with:
   - Root cause analysis (5-Whys chain)
   - Evidence from investigation
   - Proposed fix (with diff)
   - Test results
   - Confidence score
   - Action buttons: Approve / Reject / Need More Info
2. Add label `psv-im-awaiting-approval`
3. Notify approvers (SAs for code changes, TAMs for config changes)
4. Wait for approval (poll Jira comments for approval response)
5. On Approve:
   - Execute fix in target environment
   - Transition ticket to "In Review" or "Done"
   - Remove `psv-im-processing`, add `psv-im-resolved`
6. On Reject:
   - Log rejection reason
   - Save to KB (learning)
   - Optionally re-investigate with rejection feedback

### Stage 8: Reporter

**Agent**: `reporter`
**Input**: Full pipeline state
**Output**: Final Jira comment + KB entry + metrics

Logic:
1. Post final structured Jira comment (see templates)
2. Save resolution to knowledge base
3. Update metrics:
   - Time to first response (bot)
   - Time to resolution
   - Auto-resolved vs escalated
   - Fix acceptance rate
4. Remove processing labels, add resolution label
5. Update workflow state DB: `status: "resolved"`

## Workflow State

Each ticket maintains state in the DB (or state file):
```json
{
  "ticket_id": "PSV-12345",
  "pipeline": "psv-im-agent",
  "status": "processing | awaiting_approval | resolved | escalated",
  "current_stage": 4,
  "team_skill": "integration-skill",
  "retries": 0,
  "parsed_ticket": { },
  "context": { },
  "rca": { },
  "kb_matches": [ ],
  "skill_output": { },
  "qa_verdict": null,
  "approval": null,
  "timestamps": {
    "started": "...",
    "parsed": "...",
    "context_fetched": "...",
    "rca_complete": "...",
    "routed": "...",
    "skill_complete": "...",
    "qa_complete": "...",
    "approved": "...",
    "resolved": "..."
  },
  "jira_comments_posted": [ ]
}
```

## Jira Comment Format

All comments follow a structured format with headers:

```
🤖 PSV-IM Agent — [Stage Name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Content]

---
Pipeline: psv-im-agent | Stage: [N] | Ticket: [KEY]
```

## Error Handling

1. **API failure** (Jira, Confluence, Capillary) → retry 3x with exponential backoff → escalate
2. **Team skill not registered** → escalate to human immediately
3. **Team skill timeout** (>10 min) → kill, escalate to human
4. **QA fails 3x** → escalate to human
5. **Approval timeout** (>24h for P0, >72h for P1) → re-notify + escalate

## Model Selection

| Stage | Model | Why |
|---|---|---|
| Poller / Parser | haiku | Simple field extraction |
| BRD/SDD Context | sonnet | Document analysis |
| 5-Whys RCA | opus | Deep reasoning required |
| KB Correlation | sonnet | Similarity matching |
| Component Router | haiku | Simple lookup |
| QA Agent | sonnet | Validation logic |
| Approval Gate | haiku | Comment posting |
| Reporter | sonnet | Structured output |

## Configuration

Environment variables (in `.env`):
```
# Jira
JIRA_BASE_URL=https://capillarytech.atlassian.net
JIRA_EMAIL=
JIRA_API_TOKEN=
JIRA_CLOUD_ID=69031ea7-8347-4ec3-a63d-9c7289f8dc4f

# Polling
PSV_POLL_INTERVAL_MINUTES=3
PSV_POLL_PRIORITIES=P0,P1
PSV_POLL_PROJECTS=PSV

# Team Skills
REGISTERED_SKILLS=integration-skill,webapp-skill,mobile-skill,config-skill

# Approval
APPROVAL_TIMEOUT_P0_HOURS=24
APPROVAL_TIMEOUT_P1_HOURS=72

# Capillary
CAP_CONNECT_PLUS_URL=https://crm-nightly-new.connectplus.capillarytech.com
```
