# Flow design: CF_DocCentralV3_AssignPermissions

## Purpose

Assigns item-level permissions on a newly created or updated document.
Breaks SharePoint inheritance on the `Svi predmeti` item and applies explicit permissions
based on the document's organizational unit, initiator, and any approval participants.

This flow is called as a child flow by `CF_DocCentralV3_CreateDocument` after the item
and files are created. It may also be called by `CF_DocCentralV3_SendForApproval` when
a new approver gains access, and by `CF_DocCentralV3_ProcessApprovalResponse` when access
needs to be adjusted after approval completion.

## Trigger type

Child flow (called by parent flows). Not called directly from Canvas App.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Office365Groups`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_docDokumenti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

```json
{
  "sviPredmetiItemId": 0,
  "documentLibraryFileIds": [],
  "initiatorEmail": "",
  "documentType": "",
  "orgUnitGroupId": "",
  "additionalUserEmails": [],
  "additionalGroupIds": [],
  "correlationId": ""
}
```

- `sviPredmetiItemId`: SharePoint item ID in Svi predmeti — required.
- `documentLibraryFileIds`: array of SharePoint file item IDs in Dokumenti library to apply same permissions.
- `initiatorEmail`: always gets explicit access to their own document.
- `documentType`: used to look up permission rules from App Config (UNKNOWN key).
- `orgUnitGroupId`: Entra group ID for the organizational unit. Read from App Config by the caller, passed in here. Must not be hardcoded.
- `additionalUserEmails`: any individual users who should receive explicit access (e.g. approvers at the time of document creation).
- `additionalGroupIds`: any additional Entra groups to grant access (e.g. supervisors).
- `correlationId`: passed from parent flow.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Prava su uspešno dodeljena.",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Greška pri dodeli prava.",
  "errorCode": "PERMISSION_ASSIGNMENT_FAILED",
  "correlationId": ""
}
```

## Permission model (from security-model.md and architecture)

- Users have Read Only access at site/list level.
- Item-level inheritance is broken for each document item.
- After break: explicit permissions are assigned per the rules below.
- Service account always retains Full Control (it executes all writes).

### Default permission assignments per document item

| Principal | Permission level | Source |
|---|---|---|
| Service account | Full Control | Always |
| Initiator (by email) | Read | Always |
| Org unit group (`orgUnitGroupId`) | Read | Always — read from App Config |
| Additional users (`additionalUserEmails`) | Read | Passed by caller (e.g. approvers) |
| Additional groups (`additionalGroupIds`) | Read | Passed by caller |

Permission levels (Read) are defined at the SharePoint site level. The flow references the
existing site permission levels by name — it does not create new levels.

### Permission rules from App Config

The caller is responsible for reading org unit group ID and any document-type-specific
permission overrides from App Config before calling this flow. This flow does not read
App Config directly except to fall back if `orgUnitGroupId` is empty and a default group
is configured (UNKNOWN App Config key for default group).

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input `correlationId` if not empty, else `guid()`

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- EventCategory: `Permission`
- Severity: `Info`
- Status: `Started`
- DocumentItemId: input `sviPredmetiItemId`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje dodele prava na dokument."

**Step 3 — Resolve org unit group (fallback)**
If input `orgUnitGroupId` is empty:
- Read default group from App Config (UNKNOWN key for default org unit group).
- If still empty: log warning and continue without org unit group assignment.
  (Document will still have initiator access and service account access.)

**Step 4 — Break permission inheritance on Svi predmeti item**
Action: SharePoint HTTP request — Break role inheritance
Method: POST
URI: `<SharePointSite>/_api/web/lists/getByTitle('<SviPredmetiList>')/items(<sviPredmetiItemId>)/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)`

`copyRoleAssignments=false` — start with empty permissions after break.
`clearSubscopes=true` — clear any subscopings.

**Step 5 — Grant initiator access on Svi predmeti item**
Action: SharePoint HTTP request — Add role assignment for user
Resolve initiator's SharePoint user ID from email using:
`/_api/web/ensureuser('<initiatorEmail>')`
Then assign Read role using:
`/_api/web/lists/getByTitle('<SviPredmetiList>')/items(<sviPredmetiItemId>)/roleassignments/addroleassignment(principalid=<userId>,roledefid=<ReadRoleDefinitionId>)`

Read role definition ID is read from the site's role definitions (UNKNOWN — determined at runtime via `/_api/web/roledefinitions?$filter=Name eq 'Read'`). Cache this value in a variable for reuse within this flow run.

**Step 6 — Grant org unit group access on Svi predmeti item**
Condition: `orgUnitGroupId` is not empty.
Resolve group's SharePoint principal ID using `/_api/web/ensureuser('<groupId>')` or
via the group object ID from Entra (UNKNOWN — exact SharePoint group resolution method
for Entra security groups depends on the site's Entra group sync configuration).
Assign Read role to group principal.

**Step 7 — Grant additional users access on Svi predmeti item**
Apply to each item in `additionalUserEmails` array (if not empty):
- ensureuser → get user ID → addroleassignment with Read.

**Step 8 — Grant additional groups access on Svi predmeti item**
Apply to each item in `additionalGroupIds` array (if not empty):
- Resolve group principal ID → addroleassignment with Read.

**Step 9 — Apply same permissions to document library files**
Apply to each item ID in `documentLibraryFileIds` array (if not empty):
- Break role inheritance on the file item (same pattern as Step 4, targeting Dokumenti library).
- Grant initiator Read.
- Grant org unit group Read.
- Grant additional users Read.
- Grant additional groups Read.

File-level permission break uses:
`/_api/web/lists/getByTitle('<DokumentiLibrary>')/items(<fileItemId>)/breakroleinheritance(...)`

**Step 10 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: input `sviPredmetiItemId`
- CorrelationId: `varCorrelationId`
- Message: "Prava su uspešno dodeljena."

**Step 11 — Return success response**

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `PERMISSION_ASSIGNMENT_FAILED`
- ErrorMessage: error details from failed action
- CorrelationId: `varCorrelationId`

Return failure response.

## Caller responsibility

The caller (`CF_DocCentralV3_CreateDocument`, `CF_DocCentralV3_SendForApproval`, etc.)
is responsible for:
1. Looking up the correct `orgUnitGroupId` from App Config before calling this flow.
2. Passing the correct list of file IDs to protect.
3. Treating a permission failure as a non-fatal warning at document creation time
   (document is still created; permission failure is logged for manual remediation).

## Error codes

| Code | Meaning |
|---|---|
| PERMISSION_ASSIGNMENT_FAILED | General failure during permission assignment |
| PERMISSION_BREAK_FAILED | Failed to break role inheritance |
| PERMISSION_USER_RESOLVE_FAILED | Could not resolve user via ensureuser |

## Open items

| Item | Status |
|---|---|
| App Config key for default org unit group | UNKNOWN |
| App Config key for document-type-specific permission overrides | UNKNOWN |
| SharePoint Read role definition ID resolution method | To confirm — runtime lookup preferred |
| Entra security group resolution to SharePoint principal | Depends on site configuration — UNKNOWN |
| Whether file-level permissions are always applied or configurable | UNKNOWN — assume always for now |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Standard document, single org unit group | Inheritance broken, initiator + group get Read |
| No org unit group configured | Inheritance broken, initiator gets Read, warning logged |
| Additional approver passed | Approver gets Read on item and files |
| Multiple files in documentLibraryFileIds | All files get same permissions |
| SharePoint permission API returns error | Failure logged, non-fatal to document creation |
