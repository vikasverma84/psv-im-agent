# Team Skill Interface Contract

This document defines the contract that every team skill must follow to plug into the PSV-IM Agent hub.

## Overview

The PSV-IM Agent hub routes tickets to team skills based on the Jira `component` field. Each team builds and maintains their own skill. The hub provides context; the skill provides investigation, fix, and test results.

## Directory Structure

Each team skill MUST follow this structure:
```
~/.claude/skills/{skill-id}/
├── SKILL.md              # Skill metadata + orchestrator (REQUIRED)
├── agents/               # Sub-agents (REQUIRED)
│   ├── investigator.md   # Main investigation agent (REQUIRED)
│   ├── fixer.md          # Fix generation agent (OPTIONAL)
│   └── tester.md         # Test execution agent (OPTIONAL)
├── commands/
│   └── investigate.md    # Entry point command (REQUIRED)
├── references/
│   └── ...               # Team-specific references
└── settings.json
```

## SKILL.md Frontmatter (REQUIRED)

```yaml
---
name: {skill-id}              # e.g., integration-skill
description: >
  Team skill for PSV-IM Agent hub.
  Handles {component list} components.
team: {team-name}             # e.g., Integration
components:                   # PSV components this skill handles
  - Integrations
  - Data-Ingestion
capabilities:
  investigate: true           # Can perform RCA (REQUIRED)
  fix: true|false             # Can propose fixes
  test: true|false            # Can run tests
  git_access: true|false      # Needs codebase access
repos:                        # Git repos (if git_access: true)
  - name: neo-dataflows
    url: git@github.com:capillary/neo-dataflows.git
apis_used:                    # Capillary APIs this skill calls
  - debug-mode/error-events
  - debug-mode/block-stats
compatibility:
  skills:
    - psv-im-agent            # MUST declare compatibility with hub
model: sonnet
---
```

## Input Payload

The hub passes this standardized JSON to the team skill's `/investigate` command:

```json
{
  "source": "psv-im-agent",
  "version": "1.0",
  "ticket": {
    "id": "PSV-12345",
    "summary": "Points not crediting for Abbott HK",
    "description": "Full plain-text description...",
    "brand": "Abbott_HK_Prod",
    "brand_clean": "Abbott_HK",
    "component": "Integrations",
    "priority": "P0",
    "environment": "Production",
    "geo_region": "APAC",
    "org_id": "1115",
    "dataflow_id": "eace7cf2-...",
    "customer_id": "56789",
    "api_endpoint": "/v2/customers/lookup",
    "transaction_id": null,
    "request_id": null,
    "reporter": "john.doe@capillarytech.com",
    "attachments": ["screenshot.png"]
  },
  "context": {
    "sdd_content": "...",
    "brd_content": "...",
    "relevant_sections": [
      { "title": "Points Accrual", "content": "...", "relevance": 0.85 }
    ],
    "brand_config": {
      "cluster": "apac2.api.capillarytech.com",
      "org_id": "1115"
    }
  },
  "rca": {
    "symptom": "Points not crediting after transaction",
    "why_chain": [ ... ],
    "root_cause_hypothesis": "SFTP failure causing tier dataflow skip",
    "confidence": 0.55,
    "data_requests": [
      { "api": "debug-mode/error-events", "params": { "processGroupId": "..." } }
    ]
  },
  "kb_matches": [
    { "ticket_key": "PSV-11234", "resolution": "...", "similarity": 0.72 }
  ],
  "credentials": {
    "cluster_url": "https://apac2.api.capillarytech.com",
    "org_id": "1115",
    "connect_plus_url": "https://crm-nightly-new.connectplus.capillarytech.com"
  }
}
```

## Output Contract (REQUIRED)

The team skill MUST return this JSON structure:

```json
{
  "skill_id": "integration-skill",
  "skill_version": "1.0",
  "investigation": {
    "findings": "Human-readable summary of investigation",
    "evidence": [
      {
        "source": "debug-mode/error-events",
        "api_called": "POST /api/debug-mode/error-events/{id}",
        "data_summary": "3 SFTP connection errors in last 24h",
        "raw_data": { }
      }
    ],
    "root_cause_confirmed": true,
    "root_cause": "SFTP connection timeout (30s) exceeded due to partner server maintenance",
    "confidence": 0.92
  },
  "fix_proposal": {
    "type": "dataflow_config | code_change | platform_config | manual",
    "description": "Increase SFTP retry count from 1 to 3, add 60s timeout",
    "changes": [
      {
        "file": "dataflow-abbott-hk-tier.json",
        "section": "sftp-source-block",
        "before": "\"retryCount\": 1",
        "after": "\"retryCount\": 3, \"timeout\": 60000",
        "diff": "..."
      }
    ],
    "environment": "DEMO",
    "reversible": true,
    "risk_level": "low | medium | high",
    "estimated_impact": "Tier updates will resume within next scheduled run"
  },
  "test_results": {
    "smoke_passed": true,
    "tests_run": 5,
    "tests_passed": 5,
    "tests_failed": 0,
    "details": [
      { "name": "SFTP connectivity", "status": "pass", "duration_ms": 1200 },
      { "name": "Tier calculation", "status": "pass", "duration_ms": 3400 }
    ],
    "regression": false
  }
}
```

### Field Requirements

| Field | Required | Notes |
|---|---|---|
| `investigation.findings` | YES | Must be non-empty |
| `investigation.evidence` | YES | At least 1 evidence item |
| `investigation.root_cause` | YES | Must be non-empty |
| `investigation.confidence` | YES | 0.0 - 1.0 |
| `fix_proposal` | NO | Null if no fix possible — escalate |
| `fix_proposal.environment` | YES (if fix) | MUST be "DEMO" |
| `fix_proposal.reversible` | YES (if fix) | Boolean |
| `test_results` | NO | Null if skill has no test capability |
| `test_results.smoke_passed` | YES (if tests) | Boolean |

## Commands

### /investigate (REQUIRED)
Entry point called by the hub. Receives the input payload, runs investigation, returns output.

### /fix (OPTIONAL)
Separate command to apply a fix after approval. Called by the hub when human approves.

### /test (OPTIONAL)
Standalone test command. Called by hub for re-testing after fix.

## Error Response

If the skill encounters an error, return:
```json
{
  "skill_id": "integration-skill",
  "error": true,
  "error_type": "api_failure | auth_failure | timeout | unknown",
  "error_message": "Failed to connect to Connect+ debug-mode API: 401 Unauthorized",
  "partial_investigation": { ... },
  "needs_human": true
}
```

## Registration

To register a team skill with the PSV-IM Agent hub:
1. Create skill at `~/.claude/skills/{skill-id}/`
2. Include `psv-im-agent` in `compatibility.skills`
3. Implement `/investigate` command
4. Run `/team-status` to verify registration

## Example: Minimal Team Skill

```yaml
# ~/.claude/skills/config-skill/SKILL.md
---
name: config-skill
description: Config investigation skill for PSV-IM Agent. Handles Configurations and US_Configuration components.
team: Config
components:
  - Configurations
  - US_Configuration
capabilities:
  investigate: true
  fix: false
  test: false
  git_access: false
compatibility:
  skills:
    - psv-im-agent
model: sonnet
---

# Config Skill

Investigates configuration issues by checking:
1. Unified audit trail for recent config changes
2. Loyalty rule evaluation logs
3. Promotion setup validation

Does NOT propose fixes — always escalates fix to human.
```
