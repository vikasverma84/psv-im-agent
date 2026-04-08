# Capillary Platform APIs — Available for PSV-IM Agent

APIs discovered via Capillary MCP (`claude.ai Capillary Docs`). These are the data sources team skills can use for investigation.

## Neo / Connect+ Debug Mode APIs

Server: `https://crm-nightly-new.connectplus.capillarytech.com`
Auth: Session cookie + `X-CAP-API-AUTH-ORG-ID` header

### POST /api/debug-mode/error-events/{processGroupId}
**Purpose**: Get error events for a dataflow — error messages, severity, source block
**Key Params**: `?size=10000`
**Headers**: `X-CAP-API-AUTH-ORG-ID: {org_id}`, `Cookie: SESSION=...`
**Returns**:
```json
{
  "content": [{
    "eventDate": "2025-10-16T10:30:00Z",
    "processGroupId": "df-uuid",
    "errorMessage": "Connection timeout to SFTP server",
    "bulletinLevel": "ERROR",
    "sourceId": "processor-123",
    "sourceName": "SFTP Source"
  }]
}
```
**Use Case**: Find why a dataflow failed — connection errors, processing errors, block failures

### POST /api/debug-mode/block-stats/{dataflowId}
**Purpose**: Per-block input/output/failure counts
**Headers**: `X-CAP-API-AUTH-ORG-ID: {org_id}`
**Returns**:
```json
[{
  "blockId": "block1",
  "blockName": "SFTP Source",
  "inputCount": 0,
  "outputCount": 1000,
  "failureCount": 0,
  "blockOrder": 1
}]
```
**Use Case**: Pinpoint which block in a dataflow dropped records or failed

### POST /api/debug-mode/route-events
**Purpose**: Trace event routing through dataflow blocks
**Key Params**: `?processGroupId={id}&blockName={name}&size=30`
**Returns**: Content URIs, attributes (filename, record count, processing time), event dates
**Use Case**: Follow a specific record through the dataflow pipeline

### POST /api/debug-mode/jobs
**Purpose**: List jobs for a dataflow — file names, record counts, error status
**Key Params**: `?processGroupId={id}&size=100`
**Returns**:
```json
{
  "content": [{
    "dataflowId": "df-uuid",
    "lineageStartTime": "2025-10-16T10:30:00Z",
    "lineageId": "job-123",
    "fileName": "data.csv",
    "totalRecords": "1000",
    "errorFileGenerated": false,
    "totalProcessedRows": 1000
  }]
}
```
**Use Case**: Check if a dataflow job completed, how many records processed, any errors

## Dataflow Management APIs

### GET /api/v3/dataflows/{id}/versions
**Purpose**: Check recent changes to a dataflow (was it modified recently? regression?)

### POST /api/v3/dataflows/{id}/with-values
**Purpose**: Get full dataflow configuration with parameter values

### GET /api/v3/blocks
**Purpose**: List all blocks in a dataflow

### PUT /api/v3/dataflows/{id}/state/STOPPED
**Purpose**: Stop a broken dataflow (emergency)

### GET /api/v3/dataflows/{id}/publish
**Purpose**: Check publish status of a dataflow

## Audit Trail APIs

### GET /api_gateway/v2/unified-audits
**Purpose**: Comprehensive audit trail — action, status, entity, changed fields (before/after), actor
**Auth**: `Authorization: Basic {base64(till_id:md5_password)}`
**Returns**: Events with `action`, `status`, `entity.type`, `changedFields[].fieldName/before/after`, `actor`, `product`
**Use Case**: Find what changed recently — config modification, user update, rule change that could cause regression

### GET /api_gateway/v2/audits
**Purpose**: Audits filtered by entity type and reference type
**Key Params**: `?entityType={type}&referenceType={type}`

### GET /v2/events/audit_logs
**Purpose**: Event-level audit logs
**Key Params**: `?eventName={name}`

### GET /v2/events/logs
**Purpose**: Event logs by date range and event name
**Key Params**: `?from_date={date}&to_date={date}&event_name={name}`

## Customer Investigation APIs

### GET /v1.1/request/logs
**Purpose**: Request logs — CHANGE_IDENTIFIER, GOODWILL, TRANSACTION_UPDATE
**Auth**: Basic Auth
**Key Params**: `?type={type}&start_date={date}&end_date={date}&mobile={mobile}`
**Use Case**: Trace goodwill issues, identifier changes, transaction updates

### GET /v2/customers/{userId}/statusLog
**Purpose**: Customer status/tier change history
**Use Case**: Why is customer at wrong tier? When did tier change?

### GET /v2/customers/{userId}/subscriptionStatusChangeLog
**Purpose**: Subscription status changes

### POST /v2/transactions/searchEvaluationLog
**Purpose**: Transaction evaluation logs — why a rule fired or didn't fire
**Use Case**: Points not crediting? Check which promotion rules were evaluated

### GET /v2/card/statusLog
**Purpose**: Card lifecycle events (issuance, activation, deactivation)

### GET /v3/webHooks/eventLog/requestId/{requestid}
**Purpose**: Webhook delivery trace — was the webhook delivered? What was the response?

## MCP Tool Usage

To call these APIs from a skill, use Capillary MCP tools:

```
# Search for an endpoint
mcp__claude_ai_Capillary_Docs__search-endpoints(pattern: "debug-mode")

# Get endpoint details
mcp__claude_ai_Capillary_Docs__get-endpoint(path: "/api/debug-mode/jobs", method: "POST", title: "Sample API (22)")

# Execute an API call
mcp__claude_ai_Capillary_Docs__execute-request(
  title: "Sample API (22)",
  harRequest: {
    method: "POST",
    url: "https://crm-nightly-new.connectplus.capillarytech.com/api/debug-mode/jobs?processGroupId={id}&size=100",
    headers: [
      { name: "X-CAP-API-AUTH-ORG-ID", value: "{org_id}" },
      { name: "Cookie", value: "SESSION={session}" }
    ]
  }
)
```

## APIs NOT Available (Confirmed)

| System | Status | Workaround |
|---|---|---|
| MemberCare | UI-only, no REST API | Use customer GET APIs as proxy |
| DevConsole | Covered by debug-mode APIs | debug-mode/* endpoints |
| EMF Logs | Covered by debug-mode + event APIs | error-events + route-events |
