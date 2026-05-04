# Decision: Approval Mechanism

## Decision

Approval state is tracked directly in `Svi predmeti` using dedicated fields on the document item.

There is **no separate ApprovalSteps SharePoint list**.
`EV_DocCentralV3_lstApprovalSteps` does not exist and must not be created.

The approval workflow configuration (who approves, how many steps, sequential vs group) is
defined in `App Config` under **ProcesConfig** entries, loaded into `colProcesConfig` at app startup.

The Power Automate Approvals connector may be used optionally as a notification delivery
mechanism only. It is not the system of record.

## Rationale for Svi predmeti–based tracking

- Keeps all document state in one SharePoint list — simpler queries, no cross-list joins.
- Canvas App shows "documents waiting for my approval" by filtering `Svi predmeti` directly.
- No additional environment variable, list creation, or schema setup required for approval tracking.
- ProcesConfig in App Config already exists as the configuration layer — approval chain definition fits naturally there.
- Approval history stored as JSON text on the document item provides auditability without a second list.

## Confirmed Svi predmeti approval runtime fields

All display names below are **confirmed**. Internal column names are UNKNOWN until inspected.

| Display name | Type | Purpose |
|---|---|---|
| Stanje | Choice / Text | Existing field. Tracks document status including approval states. |
| InicijatorEmail | Single line of text | Email of the user who registered/submitted the document. |
| TrenutniOdobravalacEmail | Single line of text | Email of the user currently assigned to approve (empty for group steps). |
| TrenutnaGrupaOdobravanjaId | Single line of text | Entra group object ID for the current step if group-based (empty for user steps). |
| TrenutniKorakOdobravanja | Number | Which step number is currently active (1-based). |
| UkupnoKorakaOdobravanja | Number | Total number of steps in the approval chain for this document. |
| KoraciOdobravanjaJson | Multiple lines of text | JSON array of all steps with their current status (Pending/Approved/Rejected/Skipped). |
| IstorijaOdobravanjaJson | Multiple lines of text | JSON array of all completed step outcomes across all approval rounds. |

Additionally, the following archive fields are confirmed:

| Display name | Type | Purpose |
|---|---|---|
| Arhivirano | Date | Date the document was archived. |
| Arhivirao | Single line of text | Email/name of user who archived the document. |

## Approval configuration: ProcesConfig

ProcesConfig entries are stored in App Config. Exact structure is UNKNOWN until AppConfig.csv
is confirmed. The Canvas App loads ProcesConfig from App Config into `colProcesConfig` at startup.

Assumed ProcesConfig step structure (UNKNOWN — exact JSON schema):
```json
{
  "documentType": "...",
  "approvalEnabled": true,
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

ProcesConfig may be stored as a JSON string in a single App Config column,
or as multiple App Config rows — UNKNOWN until AppConfig.csv is confirmed.
All approval chain configuration is read dynamically. Nothing is hardcoded.

## KoraciOdobravanjaJson example value

```json
[
  {
    "stepNumber": 1,
    "assigneeType": "User",
    "assigneeEmail": "user1@org.rs",
    "assigneeGroupId": "",
    "status": "Approved",
    "resolvedBy": "user1@org.rs",
    "resolvedAt": "2026-05-04T10:30:00Z",
    "comments": ""
  },
  {
    "stepNumber": 2,
    "assigneeType": "Group",
    "assigneeEmail": "",
    "assigneeGroupId": "guid-of-group",
    "status": "Pending",
    "resolvedBy": null,
    "resolvedAt": null,
    "comments": null
  }
]
```

## IstorijaOdobravanjaJson example value

Accumulates entries across multiple approval rounds (after rejection and resubmission).

```json
[
  {
    "stepNumber": 1,
    "outcome": "Approved",
    "byEmail": "user1@org.rs",
    "byName": "Korisnik Jedan",
    "at": "2026-05-04T10:30:00Z",
    "comments": ""
  },
  {
    "stepNumber": 1,
    "outcome": "Rejected",
    "byEmail": "user2@org.rs",
    "byName": "Korisnik Dva",
    "at": "2026-04-10T09:15:00Z",
    "comments": "Nedostaje prilog."
  }
]
```

## Canvas App integration

The `scrZaOdobrenje` screen filters `Svi predmeti` directly:

```
Filter(
  SviPredmeti,
  Stanje = "U odobravanju" &&
  (
    TrenutniOdobravalacEmail = gblCurrentUserEmail ||
    TrenutnaGrupaOdobravanjaId in colUserGroupIds
  )
)
```

`colUserGroupIds` is loaded at app startup from the Office365Groups connector and contains
the Entra group object IDs the current user belongs to.

This filter must be delegable. `TrenutniOdobravalacEmail` and `TrenutnaGrupaOdobravanjaId`
must be indexed columns in SharePoint for performance.

For group-step filtering: if SharePoint OData delegation does not support `in` operator
for a local collection, the Canvas App constructs an explicit `Or()` filter across
the user's group IDs. If the user belongs to many groups, consider caching only
approval-relevant group IDs (from `colProcesConfig`) to limit filter size.

## Approval process rules

### Single-user approval
1. Flow reads ProcesConfig for the document type → determines step chain.
2. Flow updates `Stanje = U odobravanju`, sets `TrenutniKorakOdobravanja = 1`,
   `TrenutniOdobravalacEmail = step1.assigneeEmail`, `TrenutnaGrupaOdobravanjaId = ""`.
3. Flow serializes the step chain into `KoraciOdobravanjaJson` and writes to the document item.
4. Flow sends notification email to the approver.
5. Flow grants approver Read permission on the document item.
6. User sees document in `scrZaOdobrenje`. Clicks Approve or Reject.
7. Flow `CF_DocCentralV3_ProcessApprovalResponse` updates `KoraciOdobravanjaJson`,
   appends to `IstorijaOdobravanjaJson`, updates `Stanje` to `Odobreno` or `Odbijeno`.
8. If rejected: notifies `InicijatorEmail`. Clears `TrenutniOdobravalacEmail`.

### Sequential multi-step approval
1. All steps written to `KoraciOdobravanjaJson` at start (all Pending).
2. `TrenutniKorakOdobravanja = 1`, `TrenutniOdobravalacEmail` / `TrenutnaGrupaOdobravanjaId` = step 1 assignee.
3. On step 1 approval: step 1 marked Approved in `KoraciOdobravanjaJson`.
   `TrenutniKorakOdobravanja = 2`, current approver fields updated to step 2.
   Notification sent to step 2 assignee. Access granted.
4. Continues until final step approved → `Stanje = Odobreno`.
5. On any rejection: remaining steps marked Skipped in `KoraciOdobravanjaJson`.
   `Stanje = Odbijeno`. `TrenutniOdobravalacEmail = ""`.

### Group approval (first-to-respond wins)
1. Flow sets `TrenutnaGrupaOdobravanjaId = step.assigneeGroupId`, `TrenutniOdobravalacEmail = ""`.
2. Sends notification to all group members via Office365Groups.
3. First member to respond calls `CF_DocCentralV3_ProcessApprovalResponse`.
4. Flow reads `KoraciOdobravanjaJson` — checks if current step is still `Pending`.
5. If still Pending: accepts response, updates JSON, advances or closes process.
6. If already resolved (race condition): returns `STEP_ALREADY_RESOLVED`.

### Resubmission after rejection
Initiator may correct the document and call `CF_DocCentralV3_SendForApproval` again.
The flow re-reads ProcesConfig, resets `KoraciOdobravanjaJson`, appends a summary of
the previous round to `IstorijaOdobravanjaJson`, and restarts from step 1.

## Flow responsibilities

| Flow | Approval responsibility |
|---|---|
| CF_DocCentralV3_SendForApproval | Read ProcesConfig, write KoraciOdobravanjaJson, update Stanje and current approver fields, send notification, assign permissions |
| CF_DocCentralV3_ProcessApprovalResponse | Validate current step, update KoraciOdobravanjaJson, update Stanje and approver fields, activate next step or close, append to IstorijaOdobravanjaJson |
| CF_DocCentralV3_AssignPermissions | Grant/update item-level permissions when approver changes |

## Permission model for approval

- When a document enters approval: item-level inheritance is broken (if not already done at creation),
  and the current step's approver (user or group) is granted Read access.
- When the step advances: next approver is added to permissions. Previous approver's access
  may be retained (for audit) or revoked — per App Config retention policy (UNKNOWN key).
- Initiator always retains access.
- On rejection: permissions reset to initiator only (or per retention policy).

## Deprecation note

`PACode/sharepoint/approval-steps-list.md` is deprecated.
Do not create the ApprovalSteps list or `EV_DocCentralV3_lstApprovalSteps`.

## Open items

| Item | Status |
|---|---|
| Internal column names for all Svi predmeti approval fields | UNKNOWN — display names confirmed |
| ProcesConfig structure in App Config — exact keys and JSON format | UNKNOWN until AppConfig.csv confirmed |
| App Config key for approval permission retention policy | UNKNOWN |
| OData delegation support for TrenutnaGrupaOdobravanjaId filter in Canvas App | To validate during implementation |
