# PSV Component → Team Skill Mapping

This reference maps PSV Jira component values to team skill IDs. Used by the `component-router` agent.

## Active Mappings

| PSV Component | Team Skill ID | Team | Capability | Status |
|---|---|---|---|---|
| Integrations | `integration-skill` | Integration | investigate + fix + test | Pending |
| Data-Ingestion | `integration-skill` | Integration | investigate + fix + test | Pending |
| Webapp-CRM | `webapp-skill` | WebApp | investigate + fix + test | Pending |
| MobileApp-CRM | `mobile-skill` | Mobile App | investigate + fix + test | Pending |
| Configurations | `config-skill` | Config | investigate only | Pending |
| US_Configuration | `config-skill` | Config | investigate only | Pending |
| BAU-QA | `qa-skill` | QA | investigate + test | Pending |
| QA_task | `qa-skill` | QA | investigate + test | Pending |

## Escalation-Only Components (No Team Skill)

These components always escalate to human — no automated investigation.

| PSV Component | Escalation Target | Reason |
|---|---|---|
| Solution-Consulting | SA Team | Requires SA domain expertise |
| SI-Partner | Partner Manager | External partner involvement |
| IP_inventory | Infra Team | Infrastructure provisioning |
| self-serve-access | Platform Team | Access/permissions management |
| sushi-access | Platform Team | Access/permissions management |
| open-ports | Infra/Security | Network configuration |
| Vulnerability_Scan | Security Team | Security assessment |

## Fallback Rules

1. **Component missing** → Attempt to infer from ticket summary/description:
   - Keywords "dataflow", "Neo", "SFTP", "ETL" → `integration-skill`
   - Keywords "CRM", "webapp", "portal", "UI" → `webapp-skill`
   - Keywords "app", "mobile", "push notification" → `mobile-skill`
   - Keywords "config", "loyalty", "promotion", "tier" → `config-skill`
   - No match → ESCALATE

2. **Component = UNKNOWN** → ESCALATE

3. **Skill not registered** → ESCALATE (post Jira comment explaining why)

## Adding a New Team Skill

When a team registers their skill:
1. Update this mapping with the new skill ID
2. Change Status from "Pending" to "Active"
3. Run `/team-status` to verify

## Component Frequency (for prioritization)

Based on typical PSV ticket distribution:

| Component | % of Tickets | Priority for Skill Build |
|---|---|---|
| Integrations | ~35% | HIGH — build first |
| Configurations | ~25% | HIGH — build second |
| Data-Ingestion | ~15% | MEDIUM — covered by integration-skill |
| Webapp-CRM | ~10% | MEDIUM |
| BAU-QA | ~8% | LOW |
| MobileApp-CRM | ~3% | LOW |
| Other | ~4% | ESCALATE |
