# Flow design: CF_DocCentralV3_SendForApproval

## Purpose

Initiates the approval process for a document.
Creates one or more `ApprovalSteps` records in SharePoint.
Sends notifications to the first approver (user or group).
Updates the document `Stanje` to `U odobravanju`.
Grants the first approver(s) Read access to the document via `CF_DocCentralV3_AssignPermissions`.

This flow is called from the Canvas App by users who have the right to send a document
for approval. It does not auto-trigger — it is explicitly invoked by the initiator.

## Trigger type

Power Apps V2. Called directly from Canvas App.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Outlook`
- `gpdoccen_CR_DocCentralV3_Office365Groups`
- `gpdoccen_CR_DocCentralV3_Office365Users`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`
- `gpdoccen_EV_DocCentralV3_docDokumenti`

## ApprovalSteps list

See `PACode/sharepoint/approval-steps-list.md` for full column schema.
Environment variable for this list: UNKNOWN — `EV_DocCentralV3_lstApprovalSteps` (not yet created in solution).

## Input schema

```json
{
  "documentItemId": 0,
  "delovodniBroj": "",
  "documentType": "",
  "approvalSteps": [
    {
      "stepNumber": 1,
      "assigneeType": "User",
      "assigneeUserEmail": "",
      "assigneeGroupId": "",
      "assigneeGroupName": ""
    }
  ],
  "initiatorEmail": "",
  "initiatorDisplayName": "",
  "correlationId": ""
}
```

- `approvalSteps`: ordered array defining the full approval chain. At least one step required.
- `assigneeType`: `"User"` or `"Group"`.
- If `assigneeType = "User"`: `assigneeUserEmail` is required.
- If `assigneeType = "Group"`: `assigneeGroupId` and `assigneeGroupName` are required.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Dokument je poslat na odobrenje.",
  "firstStepAssignee": "",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Greška pri slanju na odobrenje.",
  "errorCode": "",
  "correlationId": ""
}
```

## Pre-conditions checked by flow

| Check | Failure code |
|---|---|
| Document exists in Svi predmeti | DOCUMENT_NOT_FOUND |
| Document Stanje is not already `U odobravanju` | ALREADY_IN_APPROVAL |
| Document Stanje is not `Arhivirano` | DOCUMENT_ARCHIVED |
| At least one approval step provided | NO_APPROVAL_STEPS |
| Step numbers are unique and sequential starting at 1 | INVALID_STEP_SEQUENCE |

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendForApproval`
- EventCategory: `Approval`
- Severity: `Info`
- Status: `Started`
- DocumentItemId: input `documentItemId`
- DelovodniBroj: input `delovodniBroj`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje procesa odobrenja."

**Step 3 — Read document from Svi predmeti**
Action: SharePoint — Get item
List: `EV_DocCentralV3_lstSviPredmeti`
ID: input `documentItemId`

If not found: return failure `DOCUMENT_NOT_FOUND`.

Check `Stanje` field value:
- If `U odobravanju`: return failure `ALREADY_IN_APPROVAL`.
- If `Arhivirano`: return failure `DOCUMENT_ARCHIVED`.

**Step 4 — Validate approval steps array**
Verify at least one step exists.
Verify `stepNumber` values are 1, 2, 3... without gaps or duplicates.
Verify each step has valid `assigneeType` and required assignee fields.
On any failure: return `INVALID_STEP_SEQUENCE` or `INVALID_STEP_DATA`.

**Step 5 — Create all ApprovalSteps records**
Apply to each step in `approvalSteps` input array:

Action: SharePoint — Create item
List: `EV_DocCentralV3_lstApprovalSteps`

Fields per step:

| Column | Value |
|---|---|
| Title | `concat("Odobrenje ", delovodniBroj, " - korak ", stepNumber)` |
| DocumentItemId | input `documentItemId` |
| DelovodniBroj | input `delovodniBroj` |
| StepNumber | step `stepNumber` |
| AssigneeType | step `assigneeType` |
| AssigneeUserEmail | step `assigneeUserEmail` (empty string if Group) |
| AssigneeGroupId | step `assigneeGroupId` (empty string if User) |
| AssigneeGroupName | step `assigneeGroupName` (empty string if User) |
| StepStatus | `Pending` |
| InitiatorEmail | input `initiatorEmail` |
| CorrelationId | `varCorrelationId` |
| NotificationSent | false |

Store created item IDs in a variable array `varCreatedStepIds` for reference.

**Step 6 — Update document Stanje to `U odobravanju`**
Action: SharePoint HTTP PATCH on Svi predmeti item
Update the `Stanje` column (UNKNOWN internal name) to `U odobravanju`.

If update fails: this is a critical failure. Approval steps were created but document status
not updated. Log as Error. Attempt to delete created ApprovalSteps records (compensating action).
Return failure response.

**Step 7 — Activate first step (StepNumber = 1)**
Identify the first step from the input array (`stepNumber = 1`).
Store its created SharePoint item ID as `varFirstStepItemId`.

**Step 7a — Resolve notification recipients for first step**

If `assigneeType = "User"`:
- Recipient list = [`assigneeUserEmail`]

If `assigneeType = "Group"`:
- Action: Office365Groups — List group members using `assigneeGroupId`
- Collect all member email addresses into `varGroupMemberEmails`
- Recipient list = `varGroupMemberEmails`

**Step 7b — Grant first step approver(s) access**
Call child flow `CF_DocCentralV3_AssignPermissions`:
- `sviPredmetiItemId`: input `documentItemId`
- `documentLibraryFileIds`: [] (file IDs not passed here — caller should pass if needed; default to empty for approval-only permission grant)
- `initiatorEmail`: input `initiatorEmail`
- `documentType`: input `documentType`
- `orgUnitGroupId`: empty (org unit already set at document creation; this call only adds approver)
- `additionalUserEmails`: recipient list from Step 7a (if User type or resolved group members)
- `additionalGroupIds`: [input `assigneeGroupId`] if Group type
- `correlationId`: `varCorrelationId`

Permission failure at this step is non-fatal — log warning, continue.

**Step 7c — Send notification to first step recipient(s)**
Action: Outlook — Send an email (V2) to each recipient in the list.

Email content (template — exact text UNKNOWN until localization is confirmed):
- Subject: `concat("DocCentral: Zahtev za odobrenje - ", delovodniBroj)`
- Body: Informational email describing the document, initiator, and instructions to open the Canvas App.
  - Do not include a direct approval link in the email — the approver uses the Canvas App.
  - If PA Approvals connector notification card is configured (UNKNOWN App Config toggle), also send an Approval action card.

After sending, update the first ApprovalSteps item: `NotificationSent = true`.

**Step 8 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendForApproval`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: input `documentItemId`
- DelovodniBroj: input `delovodniBroj`
- CorrelationId: `varCorrelationId`
- Message: `concat("Dokument ", delovodniBroj, " je poslat na odobrenje.")`

**Step 9 — Return success response**

### Catch scope

Compensating action attempt:
- If ApprovalSteps records were created but document status update failed:
  attempt to delete created ApprovalSteps records before returning error.

Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendForApproval`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: derived from failure point
- CorrelationId: `varCorrelationId`

Return failure response.

## Sequential approval — activation rule

Only StepNumber = 1 is activated (notification sent, permissions granted) when this flow runs.
Subsequent steps are activated by `CF_DocCentralV3_ProcessApprovalResponse` after each step completes.
All steps have `StepStatus = Pending` at creation; only the active step has `NotificationSent = true`.

## Error codes

| Code | Meaning |
|---|---|
| DOCUMENT_NOT_FOUND | Document item not found in Svi predmeti |
| ALREADY_IN_APPROVAL | Document is already in U odobravanju status |
| DOCUMENT_ARCHIVED | Document is Arhivirano and cannot enter approval |
| NO_APPROVAL_STEPS | Input approvalSteps array is empty |
| INVALID_STEP_SEQUENCE | Step numbers are not sequential or have duplicates |
| INVALID_STEP_DATA | A step is missing required assignee fields |
| STATUS_UPDATE_FAILED | Could not update document Stanje |

## Open items

| Item | Status |
|---|---|
| ApprovalSteps list environment variable | UNKNOWN — EV_DocCentralV3_lstApprovalSteps not yet in solution |
| Svi predmeti Stanje internal column name | UNKNOWN |
| App Config toggle for PA Approvals notification card | UNKNOWN |
| Email template content and language | UNKNOWN |
| Whether document library file IDs should be passed here for permission re-grant | To be decided — currently empty |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Single user approval step | One ApprovalSteps record, notification sent to user |
| Three sequential steps | Three records created, only step 1 notified |
| Group approval step | Group members resolved, all receive notification |
| Document already in U odobravanju | Failure: ALREADY_IN_APPROVAL |
| Document is Arhivirano | Failure: DOCUMENT_ARCHIVED |
| Duplicate step numbers in input | Failure: INVALID_STEP_SEQUENCE |
| Notification email fails | Warning logged, flow does not fail |
| Permission assignment fails | Warning logged, flow does not fail |
