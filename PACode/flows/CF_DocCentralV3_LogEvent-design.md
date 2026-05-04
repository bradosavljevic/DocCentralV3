# Flow design: CF_DocCentralV3_LogEvent

## Purpose

Central audit logging flow.
All other flows call this flow (or use its input schema) to write a structured audit entry to the `AuditLog` SharePoint list.

This flow must be the first implemented because all other flows depend on it for error reporting.

## Trigger type

Power Apps V2 (called by other flows via child flow pattern, or called directly from Canvas App in rare cases).

Preferred invocation: **child flow** called by other flows using the "Run a Child Flow" action.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

All fields match the AuditLog list column schema documented in `data-model/audit-log.md`.

```json
{
  "eventType": "",
  "eventCategory": "",
  "severity": "Info",
  "status": "Started",
  "correlationId": "",
  "flowName": "",
  "flowRunId": "",
  "source": "CloudFlow",
  "userEmail": "",
  "userDisplayName": "",
  "documentItemId": null,
  "delovodniBroj": "",
  "reservedNumberId": null,
  "partnerId": null,
  "message": "",
  "detailsJson": "",
  "errorCode": "",
  "errorMessage": "",
  "requestPayloadJson": "",
  "responsePayloadJson": ""
}
```

## Output schema

Success:
```json
{
  "success": true,
  "auditLogItemId": 0,
  "correlationId": ""
}
```

Failure (the logging flow itself fails):
```json
{
  "success": false,
  "message": "Audit log write failed.",
  "correlationId": ""
}
```

Note: If this flow fails to write the audit entry, the failure must not propagate and block the calling flow. The calling flow should treat audit log failure as non-fatal — it should still attempt to complete its primary operation.

## Flow steps

### 1. Compose CorrelationId
If input `correlationId` is empty, generate a new GUID as the correlation ID.
Otherwise use the provided value.

### 2. Compose CreatedAt
Set `CreatedAt` to the current UTC timestamp using `utcNow()`.

### 3. Compose AuditLog item body
Build the item body with all input fields mapped to AuditLog list columns.
Truncate `detailsJson`, `requestPayloadJson`, `responsePayloadJson` if they exceed SharePoint column limits (max 255 chars for single-line, unlimited for multi-line).
`DetailsJson`, `RequestPayloadJson`, `ResponsePayloadJson` are multi-line text columns — no truncation needed.

### 4. Create item in AuditLog list
Action: SharePoint — Create item
Site: `EV_DocCentralV3_SharePointSite`
List: `EV_DocCentralV3_lstAuditLog`
Body: composed item from step 3.

### 5. Return response
On success: return `success: true` and the new item ID.
On failure: return `success: false` with a message. Do not throw — return gracefully.

## Error handling

This flow uses a Try/Catch scope structure.

**Try scope:** Steps 1–5.

**Catch scope:**
- Compose a failure response.
- Do not attempt to re-log (would create infinite loop).
- Return failure response.

**No Finally scope needed** — this flow has no cleanup operations.

## Logging

This flow does not call itself. It is the logging endpoint. It must not create circular dependencies.

## Permission

Runs under the service account connection (`CR_DocCentralV3_SharePoint`).
Service account has Write access to the AuditLog list.

## Caller pattern

All other flows must call this flow via child flow invocation in their Catch scope at minimum.
Flows that have important milestone events (number generated, document archived, year closed) must also call this in their Try scope to record Started and Success events.

## Choice field values

EventType valid values:
```
CreateDocument
GenerateRegistryNumber
UseReservedNumber
ArchiveDocument
CloseRegistryYear
SendForApproval
ProcessApprovalResponse
SendReminder
GenerateArchiveBookPdf
PermissionAssignment
PowerAutomateError
ValidationError
SystemError
```

EventCategory valid values:
```
Document
RegistryNumber
ReservedNumber
Approval
Reminder
Archive
YearClosing
Permission
Partner
System
```

Severity valid values:
```
Info
Warning
Error
Critical
```

Status valid values:
```
Started
Success
Failed
Retried
Skipped
```

Source valid values:
```
CanvasApp
CloudFlow
SharePoint
System
```

## Test scenarios

| Scenario | Expected result |
|---|---|
| Happy path — all fields provided | AuditLog item created, item ID returned |
| Minimal fields — only required fields | AuditLog item created with nulls for optional fields |
| SharePoint write fails | Flow returns success: false, no exception thrown to caller |
| Empty correlationId input | Flow generates new GUID, uses it |
| Very large detailsJson | Written to multi-line column without truncation |
