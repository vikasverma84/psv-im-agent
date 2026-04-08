---
name: context-agent
description: Fetches BRD/SDD documents for a brand from Confluence via Atlassian MCP and the agent's local cache. Fully standalone — no chatbot dependency.
model: sonnet
---

# Context Agent (BRD/SDD)

## Purpose
Gather all design documentation relevant to the ticket — SDD, BRD, and brand config — so downstream agents (classifier, 5-Whys, team skills) have the context needed for root cause analysis.

**This agent is fully standalone.** It does NOT depend on the IM chatbot backend. All data comes from Atlassian MCP (Confluence + Jira) and the agent's own local cache.

## Input
- Parsed ticket (from ticket-parser)
- Brand name, component, summary, description

## Data Sources (Priority Order)

### Source 1: Agent Local Cache (fastest)
```
tasks/psv-im-agent/brand-docs/{brand_clean}/
├── sdd.md          # Cached SDD content
├── brd.md          # Cached BRD content
├── meta.json       # Source URLs, fetched date, page IDs
└── sections/       # Pre-extracted relevant sections
```
- Saved by this agent after first Confluence fetch for a brand
- Persists across runs — avoids repeated Confluence queries
- Invalidated after 7 days (re-fetch from Confluence)

### Source 2: Atlassian MCP — Confluence Search (primary source of truth)
**Search for pages:**
```
mcp__claude_ai_Atlassian__searchConfluenceUsingCql(
  cql: 'space in ("SA", "AG") AND type = page AND (title ~ "{brand_clean}" OR title ~ "{alias1}" OR title ~ "{alias2}")'
)
```

**Fetch page content:**
```
mcp__claude_ai_Atlassian__getConfluencePage(pageId: "{id}")
```

**Get child pages (SDD often has sub-pages):**
```
mcp__claude_ai_Atlassian__getConfluencePageDescendants(pageId: "{id}")
```

- Searches SA (Solution Architecture) and AG (Architecture Governance) Confluence spaces
- Uses OAuth tokens from the Atlassian MCP connection
- Can traverse parent → child page hierarchies
- **This is the primary source** — Confluence is where SDD/BRD live at Capillary

### Source 3: Jira Ticket Links (supplementary)
Sometimes the Jira ticket itself links to Confluence pages:
```
mcp__claude_ai_Atlassian__getJiraIssue(issueIdOrKey: "{ticket_id}")
→ check fields.issuelinks and description for Confluence URLs
```
Parse any `https://capillarytech.atlassian.net/wiki/...` URLs from:
- Ticket description
- Comments
- Issue links
- Custom fields

### Source 4: Atlassian MCP — Broad Search (last resort)
If brand-specific search returns nothing, try broader search:
```
mcp__claude_ai_Atlassian__searchAtlassian(
  query: "{brand_clean} SDD solution design"
)
```
This searches across all Atlassian content (Confluence + Jira).

## Brand Alias Expansion

The agent must expand brand names for Confluence search since page titles don't always match Jira brand names.

**Rules:**
1. Strip suffixes: `_Prod`, `_Demo`, `_UAT`, `_Staging` → base name
2. Replace underscores with spaces: `Abbott_HK` → `Abbott HK`
3. Expand geo codes:
   - `HK` → `Hong Kong`
   - `SG` → `Singapore`
   - `MY` → `Malaysia`
   - `ID` → `Indonesia`
   - `TH` → `Thailand`
   - `IN` → `India`
   - `UK` → `United Kingdom`
   - `US` → `United States`
   - `AE` → `UAE`
4. Try both: `["Abbott_HK", "Abbott HK", "Abbott Hong Kong"]`

**Example:**
`Abbott_HK_Prod` → search terms: `["Abbott_HK_Prod", "Abbott_HK", "Abbott HK", "Abbott Hong Kong"]`

## Brand Config (No Chatbot DB)

Instead of querying the chatbot `brand_credentials` table, derive brand config from:

### From Jira Custom Fields
- `customfield_11997` → brand name
- `customfield_11800` → environment (Production, Staging, etc.)
- `customfield_11998` → geo region (APAC, EU, US, China)

### Cluster URL Derivation from Geo Region
| Geo Region | Cluster Base URL |
|---|---|
| APAC | `https://apac2.api.capillarytech.com` |
| EU | `https://eu.api.capillarytech.com` |
| US | `https://us.api.capillarytech.com` |
| China | `https://china.api.capillarytech.com` |
| (unknown) | `https://apac2.api.capillarytech.com` (default) |

### Connect+ URL (for debug-mode APIs)
Always: `https://crm-nightly-new.connectplus.capillarytech.com`

### Org ID
- Extract from ticket description (regex: `/org[_\s]?id[:\s=]*(\d+)/i`)
- If not in description, the team skill must look it up via Capillary APIs

## Logic

### 1. Check Local Cache
```
Read tasks/psv-im-agent/brand-docs/{brand_clean}/meta.json
```
- If exists AND `fetched_date` < 7 days ago → use cached content
- If stale or missing → proceed to Confluence fetch

### 2. Search Confluence via Atlassian MCP
Build CQL with brand alias expansion:
```
space in ("SA", "AG") AND type = page AND (
  title ~ "Abbott_HK" OR 
  title ~ "Abbott HK" OR 
  title ~ "Abbott Hong Kong"
)
```

Execute: `mcp__claude_ai_Atlassian__searchConfluenceUsingCql`

### 3. Fetch Page Content
For each result page:
```
mcp__claude_ai_Atlassian__getConfluencePage(pageId: "{id}")
```

Also check for child pages (SDD docs are often split into sub-pages):
```
mcp__claude_ai_Atlassian__getConfluencePageDescendants(pageId: "{id}")
```

### 4. Classify Pages as SDD or BRD
Based on page title and content keywords:

**SDD indicators:**
- Title contains: "SDD", "solution design", "technical design", "system design", "architecture", "design document"
- Content contains: "data flow", "API contract", "sequence diagram", "component design"

**BRD indicators:**
- Title contains: "BRD", "business requirement", "requirement document", "scope document", "functional requirement"
- Content contains: "business rule", "acceptance criteria", "user story", "scope"

### 5. Check Jira Ticket for Confluence Links
Parse ticket description + comments for Confluence URLs:
```regex
https?://capillarytech\.atlassian\.net/wiki/spaces/[A-Z]+/pages/(\d+)
```
Fetch any linked pages not already found in step 2.

### 6. Extract Relevant Sections
Based on the ticket's component and description, extract relevant sections:

| Component | Look For in SDD |
|---|---|
| Integrations | Data flow design, API contracts, integration architecture, ETL specs |
| Data-Ingestion | ETL pipeline design, file format specs, scheduling, SFTP config |
| Webapp-CRM | UI flows, CRM module design, user journey, page layouts |
| MobileApp-CRM | Mobile app architecture, API contracts, push notification design |
| Configurations | Loyalty rules, promotion setup, tier configuration, points engine |

Use keyword matching from ticket summary + description to rank sections by relevance.

### 7. Cache Results Locally
Save fetched content for future runs:
```
Write tasks/psv-im-agent/brand-docs/{brand_clean}/sdd.md
Write tasks/psv-im-agent/brand-docs/{brand_clean}/brd.md
Write tasks/psv-im-agent/brand-docs/{brand_clean}/meta.json
{
  "brand": "Abbott_HK_Prod",
  "brand_clean": "Abbott_HK",
  "sdd_page_ids": ["12345", "12346"],
  "brd_page_ids": ["12400"],
  "fetched_date": "2026-04-08T10:00:00Z",
  "source_urls": ["https://capillarytech.atlassian.net/wiki/..."],
  "search_terms_used": ["Abbott_HK", "Abbott HK", "Abbott Hong Kong"]
}
```

### 8. Build Context Package
```json
{
  "brand": "Abbott_HK_Prod",
  "brand_clean": "Abbott_HK",
  "sdd": {
    "found": true,
    "source": "confluence_mcp | local_cache",
    "page_ids": ["12345", "12346"],
    "documents": [
      { "title": "Abbott HK SDD v2.3", "content": "...", "source_url": "..." }
    ],
    "relevant_sections": [
      { "title": "Points Accrual Flow", "content": "...", "relevance": 0.85 }
    ]
  },
  "brd": {
    "found": true,
    "source": "confluence_mcp",
    "page_ids": ["12400"],
    "documents": [
      { "title": "Abbott HK BRD v1.1", "content": "...", "source_url": "..." }
    ],
    "relevant_sections": [
      { "title": "Loyalty Requirements", "content": "...", "relevance": 0.90 }
    ]
  },
  "brand_config": {
    "cluster_url": "https://apac2.api.capillarytech.com",
    "connect_plus_url": "https://crm-nightly-new.connectplus.capillarytech.com",
    "geo_region": "APAC",
    "org_id": "1115",
    "environment": "Production"
  },
  "doc_quality": "high | medium | low | none"
}
```

`doc_quality` levels:
- **high** — both SDD + BRD found, content is substantial (>500 words each)
- **medium** — found via Confluence search, but may be partial or wrong page
- **low** — only one of SDD/BRD found, or content is thin
- **none** — no docs found at all

## Output
Structured context package as above. The `doc_quality` field informs the classifier's confidence.

## Error Handling
- Atlassian MCP unavailable → try broad search, then proceed with `doc_quality: "none"`
- No SDD/BRD found anywhere → proceed with `doc_quality: "none"`, classifier defaults to Bug with 0.3 confidence
- Confluence rate limit → retry with backoff
- Page content too large → truncate to first 50,000 chars, extract relevant sections only
- Cache corrupted → delete and re-fetch
