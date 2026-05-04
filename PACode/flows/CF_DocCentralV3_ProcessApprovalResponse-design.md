# Flow design: CF_DocCentralV3_ProcessApprovalResponse

## Purpose

Processes an approval or rejection response from an approver.
Reads the current approval state from `Svi predmeti` fields (`KoraciOdobravanjaJson`, `TrenutniKorakOdobravanja`).
Updates the current step's outcome in `KoraciOdobravanjaJson`.
Appends the decision to `IstorijaOdobravanjaJson`.
Updates document `Stanje`.
If approved and more steps remain: activates the next step (updates current approver fields, sends notification).
If rejected: sets document to `Odbijeno`, notifies `InicijatorEmail`.
If all steps approved: sets document to `Odobreno`, notifies `InicijatorEmail`.

There is no separate ApprovalSteps list. All state is on the `Svi predmeti` item.

## Trigger type

Power Apps V2. Called directly from Canvas App when an approver clicks Approve or Reject.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Outlook`
- `gpdoccen_CR_DocCentralV3_Office365Groups`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

```json
{
  "documentItemId": 0,
  "outcome": "Approved",
  "comments": "",
  "responderEmail": "",
  "responderDisplayName": "",
  "correlationId": ""
}
```

- `documentItemId`: ID of the document in Svi predmeti.
- `outcome`: `"Approved"` or `"Rejected"`.
- `responderEmail`: validated by the flow against the current step's assignee.

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

## Confirmed Svi predmeti fields read and written by this flow

| Display name | Type | Read | Written | Internal name |
|---|---|---|---|---|
| Stanje | Choice/Text | Yes | Yes | UNKNOWN |
| InicijatorEmail | Single line of text | Yes | No | UNKNOWN |
| TrenutniOdobravalacEmail | Single line of text | Yes | Yes | UNKNOWN |
| TrenutnaGrupaOdobravanjaId | Single line of text | Yes | Yes | UNKNOWN |
| TrenutniKorakOdobravanja | Number | Yes | Yes | UNKNOWN |
| UkupnoKorakaOdobravanja | Number | Yes | No | UNKNOWN |
| KoraciOdobravanjaJson | Multiple lines of text | Yes | Yes | UNKNOWN |
| IstorijaOdobravanjaJson | Multiple lines of text | Yes | Yes | UNKNOWN |

## Pre-conditions checked by flow

| Check | Failure code |
|---|---|
| Document exists in Svi predmeti | DOCUMENT_NOT_FOUND |
| Stanje is `U odobravanju` | NOT_IN_APPROVAL |
| KoraciOdobravanjaJson is not empty/null | APPROVAL_STATE_INVALID |
| Current step status is `Pending` | STEP_ALREADY_RESOLVED |
| Responder is authorized for current step | RESPONDER_NOT_AUTHORIZED |
| Outcome is `Approved` or `Rejected` | INVALID_OUTCOME |

## Authorization check

**User step** (`TrenutniOdobravalacEmail` is set):
- `responderEmail` must equal `TrenutniOdobravalacEmail` (case-insensitive).

**Group step** (`TrenutnaGrupaOdobravanjaId` is set):
- Action: Office365Groups — List members of `TrenutnaGrupaOdobravanjaId`.
- `responderEmail` must appear in the member list.
- If API fails: log warning, allow response with audit flag.

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
- DocumentItemId: input `documentItemId`
- UserEmail: input `responderEmail`
- CorrelationId: `varCorrelationId`
- Message: "Obrada odgovora odobravača."

**Step 3 — Read document from Svi predmeti**
Action: SharePoint — Get item
ID: input `documentItemId`

If not found: return failure `DOCUMENT_NOT_FOUND`.

Store:
- `varStanje` = `Stanje` value
- `varTrenutniKorak` = `TrenutniKorakOdobravanja` value
- `varUkupnoKoraka` = `UkupnoKorakaOdobravanja` value
- `varOdobravalacEmail` = `TrenutniOdobravalacEmail` value
- `varGrupaId` = `TrenutnaGrupaOdobravanjaId` value
- `varKoraciJson` = `KoraciOdobravanjaJson` value
- `varIstorijaJson` = `IstorijaOdobravanjaJson` value (default `"[]"` if null)
- `varInicijatorEmail` = `InicijatorEmail` value
- `varDelovodniBroj` = `DelovodniBroj` value
- `varDocETag` = item `@odata.etag`

**Step 4 — Validate Stanje**
If `varStanje` ≠ `U odobravanju`: return failure `NOT_IN_APPROVAL`.

**Step 5 — Validate outcome**
If `outcome` ≠ `"Approved"` and `outcome` ≠ `"Rejected"`: return failure `INVALID_OUTCOME`.

**Step 6 — Parse KoraciOdobravanjaJson**
Parse `varKoraciJson` as JSON array.
If empty, null, or unparseable: return failure `APPROVAL_STATE_INVALID`.

Find step where `stepNumber = varTrenutniKorak`.
If not found: return failure `APPROVAL_STATE_INVALID`.
Store as `varTrenutniKorakObj`.

**Step 7 — Check step is still Pending**
If `varTrenutniKorakObj.status` ≠ `"Pending"`: return failure `STEP_ALREADY_RESOLVED`.
(Race condition guard for group steps.)

**Step 8 — Authorize responder**

If `varOdobravalacEmail` is not empty (user step):
- Compare `responderEmail` to `varOdobravalacEmail` (toLower).
- If not equal: return failure `RESPONDER_NOT_AUTHORIZED`.

If `varGrupaId` is not empty (group step):
- Action: Office365Groups — List members of `varGrupaId`.
- If `responderEmail` not in member list: return failure `RESPONDER_NOT_AUTHORIZED`.
- If API fails: log warning and continue with audit flag.

**Step 9 — Update current step in KoraciOdobravanjaJson**
In the parsed JSON array, set on the current step object:
- `status` = input `outcome`
- `resolvedBy` = input `responderEmail`
- `resolvedAt` = `utcNow()`
- `comments` = input `comments`

Re-serialize to `varUpdatedKoraciJson`.

**Step 10 — Append to IstorijaOdobravanjaJson**
Parse `varIstorijaJson` as JSON array.
Append entry:
```json
{
  "stepNumber": <varTrenutniKorak>,
  "outcome": "<outcome>",
  "byEmail": "<responderEmail>",
  "byName": "<responderDisplayName>",
  "at": "<utcNow()>",
  "comments": "<comments>"
}
```
Re-serialize to `varUpdatedIstorijaJson`.

### Branch on outcome

**Step 11 — If outcome = `Rejected`**

**Step 11a — Mark remaining steps Skipped**
In `varUpdatedKoraciJson`, set `status = "Skipped"` on all steps where `stepNumber > varTrenutniKorak`.

**Step 11b — Build PATCH body**
- `Stanje` = `Odbijeno`
- `TrenutniOdobravalacEmail` = `""`
- `TrenutnaGrupaOdobravanjaId` = `""`
- `KoraciOdobravanjaJson` = updated JSON
- `IstorijaOdobravanjaJson` = updated history JSON

**Step 11c — Set `varNextAction = "DocumentRejected"`**

**Step 12 — If outcome = `Approved`**

**Step 12a — Check for next step**
Next step number = `varTrenutniKorak + 1`.
Find step in updated JSON with `stepNumber = varTrenutniKorak + 1` and `status = "Pending"`.

**Step 12b — Next step exists (sequential approval continues)**
Store `varNaredniKorak` = next step object.

Build PATCH body:
- `TrenutniKorakOdobravanja` = `varTrenutniKorak + 1`
- `TrenutniOdobravalacEmail` = `varNaredniKorak.assigneeEmail` (or `""` if group)
- `TrenutnaGrupaOdobravanjaId` = `varNaredniKorak.assigneeGroupId` (or `""` if user)
- `KoraciOdobravanjaJson` = updated JSON
- `IstorijaOdobravanjaJson` = updated history JSON
- `Stanje` = unchanged (`U odobravanju`)

**Step 12c — No next step (all steps approved)**
Build PATCH body:
- `Stanje` = `Odobreno`
- `TrenutniOdobravalacEmail` = `""`
- `TrenutnaGrupaOdobravanjaId` = `""`
- `KoraciOdobravanjaJson` = updated JSON
- `IstorijaOdobravanjaJson` = updated history JSON

Set `varNextAction = "ProcessComplete"`.

### Write outcome to Svi predmeti

**Step 13 — PATCH Svi predmeti item (If-Match)**
Action: SharePoint HTTP PATCH
Header: `If-Match: <varDocETag>`
Body: PATCH body from Step 11b, 12b, or 12c.

If 412 Precondition Failed:
- Re-read item. Check if step already resolved.
- If yes: return failure `STEP_ALREADY_RESOLVED`.
- Otherwise: retry once with fresh ETag.

If update fails: return failure `STATUS_UPDATE_FAILED`.

### Post-outcome actions

**Step 14 — If NextStepActivated: activate next step**
Resolve recipients for `varNaredniKorak`:
- User step: [`varNaredniKorak.assigneeEmail`]
- Group step: Office365Groups — List members → extract emails.

Grant next approver access via `CF_DocCentralV3_AssignPermissions` (non-fatal).
Send notification email to next step recipients (non-fatal).

**Step 15 — If ProcessComplete or DocumentRejected: notify initiator**
Action: Outlook — Send email to `varInicijatorEmail`.

If ProcessComplete:
- Subject: `concat("DocCentral: Dokument odobren - ", varDelovodniBroj)`

If DocumentRejected:
- Subject: `concat("DocCentral: Dokument odbijen - ", varDelovodniBroj)`
- Body includes: responder name, comments, instruction to review and resubmit.

Email failure is non-fatal. Log warning.

**Step 16 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ProcessApprovalResponse`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: input `documentItemId`
- DelovodniBroj: `varDelovodniBroj`
- UserEmail: input `responderEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Korak ", string(varTrenutniKorak), " - ", outcome, " od strane ", responderEmail, ". Sledeća akcija: ", varNextAction, ".")`

**Step 17 — Return success response** (include `varNextAction`)

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `ProcessApprovalResponse`
- Severity: `Error`
- Status: `Failed`
- CorrelationId: `varCorrelationId`

Return failure response.

## Stanje transition rules

| Trigger | New Stanje |
|---|---|
| Approved, next step exists | U odobravanju (unchanged) |
| Approved, no next step | Odobreno |
| Rejected | Odbijeno |

## Resubmission after rejection

Initiator calls `CF_DocCentralV3_SendForApproval` again after rejection.
`SendForApproval` validates that `Stanje = Odbijeno` is a valid starting state (allowed).
It resets `KoraciOdobravanjaJson` and appends to `IstorijaOdobravanjaJson`.

## Error codes

| Code | Meaning |
|---|---|
| DOCUMENT_NOT_FOUND | Document item not found in Svi predmeti |
| NOT_IN_APPROVAL | Document Stanje is not U odobravanju |
| APPROVAL_STATE_INVALID | KoraciOdobravanjaJson is empty, null, or unparseable |
| STEP_ALREADY_RESOLVED | Current step is no longer Pending |
| RESPONDER_NOT_AUTHORIZED | Responder is not the assigned approver |
| INVALID_OUTCOME | Outcome is not Approved or Rejected |
| STATUS_UPDATE_FAILED | Could not PATCH Svi predmeti item |

## Open items

| Item | Status |
|---|---|
| Internal column names for all Svi predmeti approval fields | UNKNOWN — display names confirmed |
| Email body templates (approval, rejection) | UNKNOWN |
| Permission retention policy for previous approvers | UNKNOWN — App Config key |

## Test scenarios

| Scenario | Expected result |
|---|---|
| User approves, single step | Stanje = Odobreno, InicijatorEmail notified |
| User approves, step 1 of 2 | Stanje unchanged, step 2 activated, step 2 notified |
| User rejects | Stanje = Odbijeno, remaining steps Skipped, InicijatorEmail notified |
| Group step, first member approves | Step resolved, next activated or process complete |
| Group step, second member tries | STEP_ALREADY_RESOLVED (ETag + KoraciOdobravanjaJson status check) |
| Wrong user tries to approve | RESPONDER_NOT_AUTHORIZED |
| Document not in U odobravanju | NOT_IN_APPROVAL |
| KoraciOdobravanjaJson null/corrupt | APPROVAL_STATE_INVALID |
| PATCH 412 conflict | Re-read, detect already resolved, return STEP_ALREADY_RESOLVED |
