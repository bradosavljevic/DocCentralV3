# Implementation steps: CF_DocCentralV3_LogEvent

## Overview

This document describes the exact build sequence for assembling the flow in Power Automate.
Follow these steps in order. Do not skip steps or reorder them.

Design reference: `PACode/flows/CF_DocCentralV3_LogEvent-design.md`
Field mapping reference: `PACode/flows/CF_DocCentralV3_LogEvent/auditlog-field-mapping.md`

---

## Pre-requisites before building

- [ ] Solution `DocCentralV3` is open in Power Apps maker portal.
- [ ] Connection reference `CR_DocCentralV3_SharePoint` is present in the solution and connected.
- [ ] Environment variable `EV_DocCentralV3_lstAuditLog` is present in the solution with a current value set to the AuditLog list name.
- [ ] Environment variable `EV_DocCentralV3_SharePointSite` is present with the SharePoint site URL.
- [ ] AuditLog SharePoint list exists with all columns from `auditlog-field-mapping.md`.

---

## Step 1 — Create the flow in the solution

1. Open solution `DocCentralV3` in Power Apps maker portal.
2. Add new → Automation → Cloud flow → Instant cloud flow.
3. Name: `CF_DocCentralV3_LogEvent`
4. Trigger: **Power Apps (V2)**
5. Save immediately before adding inputs.

---

## Step 2 — Configure trigger inputs

Add the following inputs to the Power Apps V2 trigger.
All inputs are **Text** type unless noted. In Power Automate, choose the type matching the column.

| Input name | Type | Required | Notes |
|---|---|---|---|
| eventType | Text | Yes | Choice value — pass as string |
| eventCategory | Text | Yes | Choice value — pass as string |
| severity | Text | Yes | Default: `Info` |
| status | Text | Yes | Default: `Started` |
| correlationId | Text | No | Empty string if not provided |
| flowName | Text | No | |
| flowRunId | Text | No | |
| source | Text | Yes | Default: `CloudFlow` |
| userEmail | Text | No | |
| userDisplayName | Text | No | |
| documentItemId | Number | No | Pass `0` or null if not applicable |
| delovodniBroj | Text | No | |
| reservedNumberId | Number | No | Pass `0` or null if not applicable |
| partnerId | Number | No | Pass `0` or null if not applicable |
| message | Text | Yes | |
| detailsJson | Text | No | |
| errorCode | Text | No | |
| errorMessage | Text | No | |
| requestPayloadJson | Text | No | |
| responsePayloadJson | Text | No | |

Reference: `PACode/flows/CF_DocCentralV3_LogEvent/trigger-input-schema.json`

---

## Step 3 — Add Initialize variable: varCorrelationId

Action: **Initialize variable**
- Name: `varCorrelationId`
- Type: String
- Value:
  ```
  if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])
  ```

> Note: The trigger input parameter name in the expression follows Power Automate's internal naming convention for Power Apps V2 inputs. The prefix (`text_`) may vary depending on the type. Verify in the expression editor after adding inputs.

---

## Step 4 — Add Initialize variable: varCreatedAt

Action: **Initialize variable**
- Name: `varCreatedAt`
- Type: String
- Value:
  ```
  utcNow()
  ```

---

## Step 5 — Add scope: Try

Add a **Scope** action named `Try`.
All remaining steps until the Catch scope go inside this Try scope.

Configure scope: on failure → run after failed, run after timed out (used by Catch scope below).

---

## Step 6 — Inside Try: Compose item body

Action: **Compose** (Data Operations)
Name: `Compose_AuditLogItem`

Output expression — build the JSON object that will be sent to SharePoint Create item.
This is a single Compose action whose output is the complete item body.

See `PACode/flows/CF_DocCentralV3_LogEvent/auditlog-field-mapping.md` for the full
mapping of input parameters to SharePoint column internal names.

The Compose output must be a JSON object with all mapped fields.

> Important: SharePoint Choice columns are set by passing the string value directly in the
> `Value` property of the choice field. The exact JSON structure for setting Choice fields
> in a SharePoint Create item action is:
> `"ColumnInternalName": { "Value": "<choice string>" }`
> However, using the **SharePoint — Create item** connector action (not HTTP), Power Automate
> exposes Choice columns as direct string inputs — set the field value to the choice string directly.

Key expressions per field:

```
Title          → triggerBody()?['text_eventType']
EventType      → triggerBody()?['text_eventType']
EventCategory  → triggerBody()?['text_eventCategory']
Severity       → triggerBody()?['text_severity']
Status         → triggerBody()?['text_status']
CorrelationId  → variables('varCorrelationId')
FlowName       → triggerBody()?['text_flowName']
FlowRunId      → triggerBody()?['text_flowRunId']
Source         → triggerBody()?['text_source']
UserEmail      → triggerBody()?['text_userEmail']
UserDisplayName→ triggerBody()?['text_userDisplayName']
DocumentItemId → triggerBody()?['decimal_documentItemId']
DelovodniBroj  → triggerBody()?['text_delovodniBroj']
ReservedNumberId→ triggerBody()?['decimal_reservedNumberId']
PartnerId      → triggerBody()?['decimal_partnerId']
Message        → triggerBody()?['text_message']
DetailsJson    → triggerBody()?['text_detailsJson']
ErrorCode      → triggerBody()?['text_errorCode']
ErrorMessage   → triggerBody()?['text_errorMessage']
RequestPayloadJson  → triggerBody()?['text_requestPayloadJson']
ResponsePayloadJson → triggerBody()?['text_responsePayloadJson']
CreatedAt      → variables('varCreatedAt')
EnvironmentName→ '' (static empty — or read from environment variable if available)
SolutionName   → 'DocCentralV3' (static)
```

> Note: Trigger input references (`triggerBody()?['text_*']`) use the internal name assigned
> by Power Automate to each Power Apps V2 input. The prefix type (`text_`, `decimal_`) is
> determined by the input type selected in Step 2. Verify actual names in the expression editor.

---

## Step 7 — Inside Try: SharePoint Create item

Action: **SharePoint — Create item** (using `CR_DocCentralV3_SharePoint` connection reference)
- Site Address: Environment variable `EV_DocCentralV3_SharePointSite`
  Expression: `@parameters('EV_DocCentralV3_SharePointSite')`
- List Name: Environment variable `EV_DocCentralV3_lstAuditLog`
  Expression: `@parameters('EV_DocCentralV3_lstAuditLog')`
- Map each field from the Compose output to the corresponding SharePoint column.

See `auditlog-field-mapping.md` for the complete field-to-column mapping table.

For each field, set the column value using the expression from Step 6.
Do not use the Compose output directly as the body — map fields individually in the
connector UI to ensure correct type handling.

---

## Step 8 — Inside Try: Compose success response

Action: **Compose**
Name: `Compose_SuccessResponse`

```json
{
  "success": true,
  "auditLogItemId": <ID from Create item output>,
  "correlationId": "<varCorrelationId>"
}
```

Expression:
```
{
  "success": true,
  "auditLogItemId": outputs('Create_item')?['body/ID'],
  "correlationId": variables('varCorrelationId')
}
```

---

## Step 9 — Add scope: Catch

Add a second **Scope** action named `Catch`.
Configure: Run after → has failed, has timed out (on the Try scope).
Place this scope **after** the Try scope, at the same level (not inside it).

---

## Step 10 — Inside Catch: Compose failure response

Action: **Compose**
Name: `Compose_FailureResponse`

```json
{
  "success": false,
  "message": "Audit log write failed.",
  "correlationId": "<varCorrelationId>"
}
```

Expression:
```
{
  "success": false,
  "message": "Audit log write failed.",
  "correlationId": variables('varCorrelationId')
}
```

**Do not call CF_DocCentralV3_LogEvent from within the Catch scope.**
That would create an infinite loop.

---

## Step 11 — Add Respond to a PowerApp or flow (success path)

Add this action **after** the Catch scope, at the root level, configured to run after:
- Try scope succeeds → respond with `Compose_SuccessResponse` output
- Catch scope runs → respond with `Compose_FailureResponse` output

Implementation approach — use a **Switch** or **Condition** on whether the Try scope
succeeded, then respond with the appropriate Compose output:

Pattern:
1. Add **Condition** after Catch scope: check if Try scope result is `Succeeded`.
   Expression: `result('Try')?[0]?['status']` equals `Succeeded`
2. Yes branch: **Respond to a PowerApp or flow** with `Compose_SuccessResponse` output.
3. No branch: **Respond to a PowerApp or flow** with `Compose_FailureResponse` output.

Both response actions must return the same output parameter names:
- `success` (Boolean or Text)
- `auditLogItemId` (Number — return `0` in failure branch)
- `correlationId` (Text)
- `message` (Text — return empty string in success branch)

Reference: `PACode/flows/CF_DocCentralV3_LogEvent/response-schema.json`

---

## Step 12 — Configure run settings

- Concurrency control: off (no concurrency limit needed for a logging flow).
- Timeout: default (no change).
- Connection reference: ensure all SharePoint actions use `CR_DocCentralV3_SharePoint`.

---

## Step 13 — Save and verify

1. Save the flow.
2. Check for any missing required fields or expression errors.
3. Verify the flow appears in the `DocCentralV3` solution.
4. Do not test yet — see `test-cases.md`.

---

## Step 14 — Add to solution

If the flow was not created inside the solution:
1. Open solution `DocCentralV3`.
2. Add existing → Automation → Cloud flow → select `CF_DocCentralV3_LogEvent`.
3. Confirm it appears under Automation in the solution.

---

## Notes for Power Automate builder

- The Power Apps V2 trigger generates internal input names automatically. Always verify
  actual internal names in the expression editor before saving expressions.
- Use dynamic content picker where possible to reduce expression errors.
- The `CR_DocCentralV3_SharePoint` connection reference must be selected (not a personal connection)
  in every SharePoint action.
- If the AuditLog list column for `CreatedAt` is read-only (set by SharePoint automatically),
  use the `Modified` or `Created` system columns for time reference, or keep `CreatedAt` as
  a custom column distinct from the system `Created` column.
