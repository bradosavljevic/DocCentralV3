# Flow design: CF_DocCentralV3_SendForApproval

## Purpose

Initiates the approval process for a document.
Reads the approval chain from **ProcesConfig** in App Config.
Writes the approval chain definition and current step state to `Svi predmeti` fields.
Sends a notification email to the first step's approver.
Updates the document `Stanje` to `U odobravanju`.
Grants the first approver(s) Read access via `CF_DocCentralV3_AssignPermissions`.

There is no separate ApprovalSteps SharePoint list.
All approval state lives on the `Svi predmeti` item.

## Trigger type

Power Apps V2. Called directly from Canvas App.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Outlook`
- `gpdoccen_CR_DocCentralV3_Office365Groups`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

```json
{
  "documentItemId": 0,
  "delovodniBroj": "",
  "documentType": "",
  "initiatorEmail": "",
  "initiatorDisplayName": "",
  "correlationId": ""
}
```

The approval chain is **not passed by the Canvas App**.
It is read from ProcesConfig in App Config by the flow, preventing Canvas App override.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Dokument je poslat na odobrenje.",
  "totalSteps": 0,
  "firstApprover": "",
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

## Confirmed Svi predmeti fields written by this flow

| Display name | Type | Internal name | Value set |
|---|---|---|---|
| Stanje | Choice/Text | UNKNOWN | `U odobravanju` |
| TrenutniKorakOdobravanja | Number | UNKNOWN | `1` |
| TrenutniOdobravalacEmail | Single line of text | UNKNOWN | Step 1 user email (or `""` if group) |
| TrenutnaGrupaOdobravanjaId | Single line of text | UNKNOWN | Step 1 group ID (or `""` if user) |
| UkupnoKorakaOdobravanja | Number | UNKNOWN | Count of steps from ProcesConfig |
| KoraciOdobravanjaJson | Multiple lines of text | UNKNOWN | Serialized JSON — all steps with status `Pending` |
| IstorijaOdobravanjaJson | Multiple lines of text | UNKNOWN | `[]` on first send; preserved + extended on resubmission |

## ProcesConfig dependency

Flow reads ProcesConfig from App Config filtered by `documentType` (UNKNOWN filter key).
If no ProcesConfig found for the document type: try a default/fallback config.
If none: fail with `NO_PROCESS_CONFIG`.

Assumed ProcesConfig step structure (UNKNOWN until AppConfig.csv confirmed):
```json
{
  "steps": [
    {
      "stepNumber": 1,
      "assigneeType": "User",
      "assigneeEmail": "approver@org.rs",
      "assigneeGroupId": ""
    }
  ]
}
```

## Pre-conditions checked by flow

| Check | Failure code |
|---|---|
| Document exists in Svi predmeti | DOCUMENT_NOT_FOUND |
| Stanje is not `U odobravanju` | ALREADY_IN_APPROVAL |
| Stanje is not `Arhivirano` | DOCUMENT_ARCHIVED |
| ProcesConfig exists for document type | NO_PROCESS_CONFIG |
| ProcesConfig has at least one step | NO_APPROVAL_STEPS |

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varApprovalSteps` (Array) = []

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
ID: input `documentItemId`

If not found: return failure `DOCUMENT_NOT_FOUND`.

Check `Stanje`:
- If `U odobravanju`: return failure `ALREADY_IN_APPROVAL`.
- If `Arhivirano`: return failure `DOCUMENT_ARCHIVED`.

Store `varDocETag` = item `@odata.etag`.
Store existing `IstorijaOdobravanjaJson` value for resubmission handling (Step 6).

**Step 4 — Read ProcesConfig from App Config**
Action: SharePoint — Get items from App Config
Filter: ProcesConfig category AND documentType matches input `documentType` (UNKNOWN filter expression).
Top: 1

If not found: try default ProcesConfig (UNKNOWN key).
If still not found: return failure `NO_PROCESS_CONFIG`.

Parse the steps JSON from ProcesConfig into `varApprovalSteps` array.
If empty: return failure `NO_APPROVAL_STEPS`.

**Step 5 — Build initial KoraciOdobravanjaJson**
Compose JSON array from `varApprovalSteps` with `status = "Pending"` on each step.

Example:
```json
[
  {"stepNumber": 1, "assigneeType": "User", "assigneeEmail": "a@b.rs", "assigneeGroupId": "", "status": "Pending", "resolvedBy": null, "resolvedAt": null, "comments": null},
  {"stepNumber": 2, "assigneeType": "Group", "assigneeEmail": "", "assigneeGroupId": "group-guid", "status": "Pending", "resolvedBy": null, "resolvedAt": null, "comments": null}
]
```

Store as `varKoraciJson` string.

**Step 6 — Build or preserve IstorijaOdobravanjaJson**
If existing `IstorijaOdobravanjaJson` from Step 3 is not empty and not `[]`:
- This is a resubmission. Append a round-marker entry to the existing history.
- Store as `varIstorijaJson`.
Otherwise:
- `varIstorijaJson = "[]"`.

**Step 7 — Update Svi predmeti item with approval state (If-Match)**
Action: SharePoint HTTP PATCH on Svi predmeti item
Header: `If-Match: <varDocETag>`

Fields (display names confirmed; internal names UNKNOWN — replace with internal names when confirmed):
- `Stanje` → `U odobravanju`
- `TrenutniKorakOdobravanja` → `1`
- `TrenutniOdobravalacEmail` → step 1 `assigneeEmail` (or `""` if group)
- `TrenutnaGrupaOdobravanjaId` → step 1 `assigneeGroupId` (or `""` if user)
- `UkupnoKorakaOdobravanja` → count of steps in `varApprovalSteps`
- `KoraciOdobravanjaJson` → `varKoraciJson`
- `IstorijaOdobravanjaJson` → `varIstorijaJson`

If 412 Precondition Failed: re-read item, re-check Stanje. Retry once with fresh ETag.
If update fails: return failure `STATUS_UPDATE_FAILED`.

**Step 8 — Resolve notification recipients for step 1**
Store step 1 from `varApprovalSteps` as `varStep1`.

If `varStep1.assigneeType = "User"`:
- `varRecipients` = [`varStep1.assigneeEmail`]

If `varStep1.assigneeType = "Group"`:
- Action: Office365Groups — List members of `varStep1.assigneeGroupId`
- Extract member emails → `varRecipients`

**Step 9 — Grant step 1 approver(s) access**
Call child flow `CF_DocCentralV3_AssignPermissions`:
- `sviPredmetiItemId`: input `documentItemId`
- `documentLibraryFileIds`: []
- `initiatorEmail`: input `initiatorEmail`
- `documentType`: input `documentType`
- `orgUnitGroupId`: empty (already set at document creation)
- `additionalUserEmails`: `varRecipients` if User type, else []
- `additionalGroupIds`: [`varStep1.assigneeGroupId`] if Group type, else []
- `correlationId`: `varCorrelationId`

Permission failure is non-fatal. Log warning and continue.

**Step 10 — Send notification email to step 1 recipients**
For each recipient in `varRecipients`:
Action: Outlook — Send email (V2)
- Subject: `concat("DocCentral: Zahtev za odobrenje dokumenta ", delovodniBroj)`
- Body: Document registry number, initiator name, instruction to open Canvas App.
  (Template UNKNOWN — placeholder until confirmed.)
- Approver uses Canvas App. No approval links in email.

Email failure per recipient is non-fatal. Log warning.

**Step 11 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendForApproval`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: input `documentItemId`
- DelovodniBroj: input `delovodniBroj`
- CorrelationId: `varCorrelationId`
- Message: `concat("Dokument ", delovodniBroj, " je poslat na odobrenje. Koraci: ", string(length(varApprovalSteps)), ".")`

**Step 12 — Return success response**

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendForApproval`
- Severity: `Error`
- Status: `Failed`
- CorrelationId: `varCorrelationId`

Return failure response.

## Error codes

| Code | Meaning |
|---|---|
| DOCUMENT_NOT_FOUND | Document item not found in Svi predmeti |
| ALREADY_IN_APPROVAL | Document Stanje is already U odobravanju |
| DOCUMENT_ARCHIVED | Document is Arhivirano, cannot enter approval |
| NO_PROCESS_CONFIG | No ProcesConfig found in App Config for this document type |
| NO_APPROVAL_STEPS | ProcesConfig has no steps defined |
| STATUS_UPDATE_FAILED | Could not PATCH Svi predmeti with approval state |

## Open items

| Item | Status |
|---|---|
| Internal column names for all Svi predmeti approval fields | UNKNOWN — display names confirmed |
| ProcesConfig App Config structure and exact filter key | UNKNOWN until AppConfig.csv confirmed |
| Notification email body template | UNKNOWN |
| Approval permission retention policy App Config key | UNKNOWN |
| Resubmission detection — whether IstorijaOdobravanjaJson = null or [] on first send | To confirm from actual list default value |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Single user step, ProcesConfig found | Stanje = U odobravanju, step 1 fields set, email sent |
| Two sequential steps | Both steps in KoraciOdobravanjaJson, only step 1 activated |
| Group step | Group members resolved, all notified, TrenutnaGrupaOdobravanjaId set |
| Document already in U odobravanju | ALREADY_IN_APPROVAL |
| No ProcesConfig for document type | NO_PROCESS_CONFIG |
| Email notification fails | Warning logged, flow succeeds |
| Permission grant fails | Warning logged, flow succeeds |
| Concurrent update (412 on Svi predmeti) | Retry once with fresh ETag |
| Resubmission after rejection | IstorijaOdobravanjaJson preserved, KoraciOdobravanjaJson reset |
