---
description: Show registered team skills, their component mappings, and health status.
argument-hint: [--verbose]
---

# /team-status — Team Skill Registry

## Usage
```
/team-status              # Show registered skills summary
/team-status --verbose    # Show full details including capabilities
```

## Logic

### 1. Scan Skill Registry
Check each expected team skill directory:
```
~/.claude/skills/integration-skill/SKILL.md
~/.claude/skills/webapp-skill/SKILL.md
~/.claude/skills/mobile-skill/SKILL.md
~/.claude/skills/config-skill/SKILL.md
~/.claude/skills/qa-skill/SKILL.md
```

### 2. Read Skill Manifests
For each found skill, read the SKILL.md frontmatter to extract:
- `name`
- `description`
- `team`
- `components` (which PSV components it handles)
- `capabilities` (investigate, fix, test)

### 3. Display Summary
```
🤖 PSV-IM Agent — Team Skill Registry
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REGISTERED SKILLS:
┌───────────────────┬──────────────────────────┬─────────────────┬─────────┐
│ Skill             │ Components               │ Capabilities    │ Status  │
├───────────────────┼──────────────────────────┼─────────────────┼─────────┤
│ integration-skill │ Integrations,            │ ✅ investigate  │ ✅ OK   │
│                   │ Data-Ingestion           │ ✅ fix          │         │
│                   │                          │ ✅ test         │         │
├───────────────────┼──────────────────────────┼─────────────────┼─────────┤
│ webapp-skill      │ Webapp-CRM               │ ✅ investigate  │ ❌ NOT  │
│                   │                          │ ❌ fix          │ FOUND   │
│                   │                          │ ❌ test         │         │
├───────────────────┼──────────────────────────┼─────────────────┼─────────┤
│ mobile-skill      │ MobileApp-CRM            │ ❌ not found    │ ❌ NOT  │
│                   │                          │                 │ FOUND   │
├───────────────────┼──────────────────────────┼─────────────────┼─────────┤
│ config-skill      │ Configurations,          │ ❌ not found    │ ❌ NOT  │
│                   │ US_Configuration         │                 │ FOUND   │
└───────────────────┴──────────────────────────┴─────────────────┴─────────┘

UNHANDLED COMPONENTS (will escalate to human):
  • Solution-Consulting
  • SI-Partner
  • IP_inventory
  • self-serve-access
  • sushi-access
  • open-ports
  • Vulnerability_Scan

COVERAGE: 2/15 components covered (13%)
```

### 4. Verbose Mode (--verbose)
For each registered skill, also show:
- Git repos configured
- APIs used
- Last invocation timestamp
- Success rate (from metrics)
- Average resolution time

### 5. Health Check
For each registered skill:
1. Verify SKILL.md exists and parses correctly
2. Check required commands exist (`/investigate` at minimum)
3. Check dependencies (MCP tools, Git repos)
4. Report status: OK / DEGRADED / NOT FOUND
