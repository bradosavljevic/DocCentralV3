# Flow design: CF_DocCentralV3_ProcessApprovalResponse

## Purpose

Processes an approval or rejection response from an approver.
Updates the `ApprovalSteps` record with the outcome.
Updates the document `Stanje` in `Svi predmeti`.
If approved and more steps remain, activates the next step.
If rejected, sets document to `Odbijeno` and notifies the initiator.
If all steps are approved, sets document to `Odobreno`.

This flow is the resolution point for the entire approval process.

## Trigger type

Power Apps V2. Called directly from Canvas App when an approver clicks Approve or Reject.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Outlook`
- `gpdoccen_CR_DocCentralV3_Office365Groups`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`
- `gpdoccen_EV_DocCentralV3_docDokumenti`

ApprovalSteps list: environment variable UNKNOWN (`EV_DocCentralV3_lstApprovalSteps`).

## Input schema

```json
{
  "approvalStepItemId": 0,
  "outcome": "Approved",
  "comments": "",
  "responderEmail": "",
  "responderDisplayName": "",
  "correlationId": ""
}
```

- `approvalStepItemId`: ID of the specific `ApprovalSteps` item being resolved.
- `outcome`: `"Approved"` or `"Rejected"`.
- `responderEmail`: email of the user who is responding (validated against step's assignee).
- `comments`: optional free-text comment from the approver.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Odgovor je zabeležen.",
  "nextAction": "NextStepActivated | ProcessComplete | DocumentRejected",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Greška pri obradi odgovora.",
  "errorCode": "",
  "correlationId": ""
}
```

## Pre-conditions checked by flow

| Check | Failure code |
|---|---|
| ApprovalSteps item exists | STEP_NOT_FOUND |
| Step StepStatus is `Pending` | STEP_ALREADY_RESOLVED |
| Responder is authorized for this step | RESPONDER_NOT_AUTHORIZED |
| Outcome value is `Approved` or `Rejected` | INVALID_OUTCOME |

## Authorization check

### User step
- `assigneeType = "User"`: `responderEmail` must equal `assigneeUserEmail` on the step record.

### Group step
- `assigneeType = "Group"`: verify that `responderEmail` is a member of `assigneeGroupId`.
- Action: Office365Groups — List group members. Check that `responderEmail` appears in results.
- If member check fails (API error): log warning and fall back to allowing response (non-blocking).
  Flag in audit log for manual review.

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varNextAction` (String) = empty

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ProcessApprovalResponse`
- EventCategory: `Approval`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `responderEmail`
- CorrelationId: `varCorrelationId`
- Message: "Obrada odgovora odobravača."

**Step 3 — Read ApprovalSteps item**
Action: SharePoint — Get item
List: `EV_DocCentralV3_lstApprovalSteps`
ID: input `approvalStepItemId`

If not found: return failure `STEP_NOT_FOUND`.

Store:
- `varDocumentItemId` = item `DocumentItemId`
- `varDelovodniBroj` = item `DelovodniBroj`
- `varStepNumber` = item `StepNumber`
- `varStepStatus` = item `StepStatus`
- `varAssigneeType` = item `AssigneeType`
- `varAssigneeUserEmail` = item `AssigneeUserEmail`
- `varAssigneeGroupId` = item `AssigneeGroupId`
- `varInitiatorEmail` = item `InitiatorEmail`
- `varStepETag` = item `@odata.etag`

**Step 4 — Check step is still Pending**
If `varStepStatus` is not `Pending`: return failure `STEP_ALREADY_RESOLVED`.
(Race condition guard: two group members respond simultaneously. The second one is rejected here.)

**Step 5 — Validate outcome**
If `outcome` is not `"Approved"` or `"Rejected"`: return failure `INVALID_OUTCOME`.

**Step 6 — Authorize responder**

If `varAssigneeType = "User"`:
- Check `responderEmail == varAssigneeUserEmail` (case-insensitive).
- If not equal: return failure `RESPONDER_NOT_AUTHORIZED`.

If `varAssigneeType = "Group"`:
- Action: Office365Groups — List members of `varAssigneeGroupId`.
- Check `responderEmail` appears in member list.
- If not found: return failure `RESPONDER_NOT_AUTHORIZED`.
- If group member list API call fails: log warning, allow response, flag in audit.

**Step 7 — Mark ApprovalSteps item as resolved (If-Match for race safety)**
Action: SharePoint HTTP PATCH on ApprovalSteps item
Header: `If-Match: <varStepETag>`

Update fields:
- `StepStatus` = input `outcome` (`Approved` or `Rejected`)
- `OutcomeByEmail` = input `responderEmail`
- `OutcomeByDisplayName` = input `responderDisplayName`
- `OutcomeAt` = `utcNow()`
- `Comments` = input `comments`

If 412 Precondition Failed: another responder resolved this step simultaneously.
Return failure `STEP_ALREADY_RESOLVED`.

### Branch on outcome

**Step 8 — If outcome = `Rejected`**

**Step 8a — Update document Stanje to `Odbijeno`**
Action: SharePoint PATCH on Svi predmeti item `varDocumentItemId`
Set `Stanje` = `Odbijeno` (UNKNOWN internal name).

**Step 8b — Mark remaining Pending steps as Skipped**
Action: SharePoint — Get items from ApprovalSteps where `DocumentItemId = varDocumentItemId` AND `StepStatus = Pending`.
For each: PATCH to set `StepStatus = Skipped`.

**Step 8c — Notify initiator of rejection**
Action: Outlook — Send email to `varInitiatorEmail`.
Subject: `concat("DocCentral: Dokument odbijen - ", varDelovodniBroj)`
Body: Include `responderDisplayName`, `comments`, and instruction to review and resubmit.

**Step 8d — Set `varNextAction` = `"DocumentRejected"`**

**Step 9 — If outcome = `Approved`**

**Step 9a — Look for next pending step**
Action: SharePoint — Get items from ApprovalSteps
Filter: `DocumentItemId = varDocumentItemId` AND `StepNumber = (varStepNumber + 1)` AND `StepStatus = Pending`
Top: 1

**Step 9b — If next step found: activate it**

Resolve notification recipients (same logic as `CF_DocCentralV3_SendForApproval` Step 7a):
- If User: recipient = [`nextStep.AssigneeUserEmail`]
- If Group: resolve group members

Grant next approver(s) access:
Call `CF_DocCentralV3_AssignPermissions`:
- `additionalUserEmails` or `additionalGroupIds` set to next step's assignee
- `sviPredmetiItemId`: `varDocumentItemId`
- other fields as appropriate

Send notification email to next step recipients (same template as SendForApproval).

Update next step item: `NotificationSent = true`.

Set `varNextAction` = `"NextStepActivated"`.

**Step 9c — If no next step found: all steps approved**

Update document `Stanje` = `Odobreno` (UNKNOWN internal name).

Notify initiator of full approval:
Action: Outlook — Send email to `varInitiatorEmail`.
Subject: `concat("DocCentral: Dokument odobren - ", varDelovodniBroj)`

Set `varNextAction` = `"ProcessComplete"`.

### Finalization

**Step 10 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ProcessApprovalResponse`
- EventCategory: `Approval`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: `varDocumentItemId`
- DelovodniBroj: `varDelovodniBroj`
- UserEmail: input `responderEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Odgovor ", outcome, " zabeležen za korak ", string(varStepNumber), " dokumenta ", varDelovodniBroj, ".")`

**Step 11 — Return success response**
Include `varNextAction` in response.

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `ProcessApprovalResponse`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: derived from failure point
- CorrelationId: `varCorrelationId`

Return failure response.

## Stanje transition rules

| Current Stanje | Outcome | New Stanje |
|---|---|---|
| U odobravanju | Approved + more steps remain | U odobravanju (unchanged) |
| U odobravanju | Approved + no more steps | Odobreno |
| U odobravanju | Rejected | Odbijeno |

## After rejection — resubmit path (Canvas App responsibility)

After rejection:
- Initiator receives notification.
- Initiator may edit document metadata in Canvas App.
- Initiator may call `CF_DocCentralV3_SendForApproval` again.
- `SendForApproval` validates that `Stanje = Odbijeno` is an allowed starting state for a new approval (not blocked, unlike `Arhivirano`).

## Error codes

| Code | Meaning |
|---|---|
| STEP_NOT_FOUND | ApprovalSteps item does not exist |
| STEP_ALREADY_RESOLVED | Step is no longer Pending (race condition) |
| RESPONDER_NOT_AUTHORIZED | Responder is not the assigned approver for this step |
| INVALID_OUTCOME | Outcome value is not Approved or Rejected |
| STATUS_UPDATE_FAILED | Could not update Svi predmeti Stanje |

## Open items

| Item | Status |
|---|---|
| ApprovalSteps list EV logical name | UNKNOWN |
| Svi predmeti Stanje internal column name | UNKNOWN |
| ApprovalSteps StepStatus internal column name | UNKNOWN |
| Email body templates | UNKNOWN |
| Whether permission is revoked from rejected approver | UNKNOWN — decide: revoke Read or retain for audit |
| Whether Odbijeno → SendForApproval is explicitly allowed | To confirm from process docs — assumed yes |

## Test scenarios

| Scenario | Expected result |
|---|---|
| User approves, single step | Stanje = Odobreno, initiator notified |
| User approves, step 1 of 2 | Stanje unchanged, step 2 activated |
| User rejects | Stanje = Odbijeno, remaining steps Skipped, initiator notified |
| Group step, first member approves | Step resolved, next step activated or process complete |
| Group step, second member tries to approve | STEP_ALREADY_RESOLVED returned |
| Wrong user tries to approve | RESPONDER_NOT_AUTHORIZED |
| ApprovalSteps item not found | STEP_NOT_FOUND |
| Race condition on ETag update | 412 handled as STEP_ALREADY_RESOLVED |
