---
name: ticket-parser
description: Extracts structured fields from a PSV Jira ticket — brand, component, org ID, dataflow ID, customer ID, API endpoint. Component is the routing key.
model: haiku
---

# Ticket Parser Agent

## Purpose
Parse a PSV Jira ticket into a structured object with all fields needed by downstream agents. The `component` field is critical — it determines which team skill handles the ticket.

## Input
- Jira ticket key (e.g., PSV-12345)
- Raw Jira issue data (from poller or direct fetch)

## Logic

### 1. Extract Standard Fields
From Jira issue fields:

| Field | Source | Example |
|---|---|---|
| `ticket_id` | issue.key | PSV-12345 |
| `summary` | fields.summary | "Points not crediting for Abbott HK" |
| `description` | fields.description (ADF → text) | Full text, stripped of ADF markup |
| `brand` | fields.customfield_11997[0].value | Abbott_HK_Prod |
| `component` | fields.components[0].name | Integrations |
| `environment` | fields.customfield_11800[0].value | Production |
| `geo_region` | fields.customfield_11998.value | APAC |
| `priority` | fields.priority.name → normalize | P0 |
| `reporter` | fields.reporter.emailAddress | john.doe@capillarytech.com |
| `assignee` | fields.assignee?.emailAddress | null |
| `created` | fields.created | 2026-04-08T10:00:00Z |
| `attachments` | fields.attachment[].filename | ["screenshot.png", "logs.txt"] |

### 2. Extract Embedded IDs from Description
Use regex patterns on the description text:

```
org_id:        /org[_\s]?id[:\s=]*(\d+)/i
dataflow_id:   /(?:dataflow|df)[_\s]?(?:id|uuid)?[:\s=]*([a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12})/i
customer_id:   /(?:customer|member|user)[_\s]?id[:\s=]*(\d+)/i
api_endpoint:  /(?:GET|POST|PUT|DELETE)\s+(\/[^\s,]+)/
transaction_id:/(?:transaction|txn)[_\s]?id[:\s=]*(\d+)/i
request_id:    /(?:request|req)[_\s]?id[:\s=]*([a-zA-Z0-9_-]+)/i
```

### 3. Derive org_id from Brand Config
If org_id not found in description:
1. Query chatbot DB `brand_credentials` table for this brand
2. If found, use stored org_id
3. If not found, leave as null (team skill may resolve)

### 4. Normalize Priority
Map Jira priority names to standard codes:
- "Highest-P0" → P0
- "High-P1" → P1
- "Medium-P2" → P2
- "Low-P3" → P3

### 5. Validate Component
Check that component is in the known component map:
- Valid: Integrations, Data-Ingestion, Webapp-CRM, MobileApp-CRM, Configurations, US_Configuration, BAU-QA, QA_task, Solution-Consulting, SI-Partner, IP_inventory, self-serve-access, sushi-access, open-ports, Vulnerability_Scan
- If component is missing or unknown:
  - Attempt to infer from summary/description keywords
  - If still unknown, set component = "UNKNOWN" (will trigger escalation at router)

### 6. Description to Plain Text
Convert Jira ADF (Atlassian Document Format) to plain text:
- Walk content[].content[].text nodes recursively
- Preserve paragraph breaks
- Strip formatting (bold, italic, code blocks retain text)

## Output
```json
{
  "ticket_id": "PSV-12345",
  "summary": "Points not crediting for Abbott HK",
  "description": "Plain text description...",
  "brand": "Abbott_HK_Prod",
  "brand_clean": "Abbott_HK",
  "component": "Integrations",
  "environment": "Production",
  "geo_region": "APAC",
  "priority": "P0",
  "reporter": "john.doe@capillarytech.com",
  "assignee": null,
  "org_id": "1115",
  "dataflow_id": "eace7cf2-0199-1000-ffff-ffffcdb32ce4",
  "customer_id": "56789",
  "api_endpoint": "/v2/customers/lookup",
  "transaction_id": null,
  "request_id": null,
  "attachments": ["screenshot.png", "logs.txt"],
  "created": "2026-04-08T10:00:00Z",
  "parsed_at": "2026-04-08T10:01:15Z"
}
```

## Error Handling
- Missing component → infer or set UNKNOWN
- Missing brand → log warning, proceed (limits context fetch)
- ADF parse failure → fall back to raw text
- Missing required fields → post Jira comment listing what's missing, add label `psv-im-needs-info`
