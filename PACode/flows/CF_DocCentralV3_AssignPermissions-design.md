# Flow design: CF_DocCentralV3_AssignPermissions

## Purpose

Assigns item-level permissions on a document item and optionally on its library files.
Breaks SharePoint inheritance on the `Svi predmeti` item and applies explicit permissions
based on the document's organizational unit, initiator, and any approval participants.

This flow is called as a child flow by:
- `CF_DocCentralV3_CreateDocument` — after item and files are created.
- `CF_DocCentralV3_SendForApproval` — to grant the first step's approver access.
- `CF_DocCentralV3_ProcessApprovalResponse` — to grant access to the next step's approver when a step advances.

There is no ApprovalSteps list. Permission grants for approval are driven by the current step
state stored on the `Svi predmeti` item.

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
- `documentLibraryFileIds`: array of SharePoint file item IDs in Dokumenti library to apply same permissions. May be empty.
- `initiatorEmail`: always gets explicit access to their own document.
- `documentType`: used to look up permission rules from App Config (UNKNOWN key).
- `orgUnitGroupId`: Entra group ID for the organizational unit. Read from App Config by the caller and passed here. Must not be hardcoded.
- `additionalUserEmails`: individual users to grant access (e.g. approver at current step).
- `additionalGroupIds`: additional Entra groups to grant access (e.g. approver group at current step).
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

## Permission model

- Users have Read Only access at site/list level.
- Item-level inheritance is broken per document item.
- After break: explicit permissions assigned per rules below.
- Service account always retains Full Control.

### Default permission assignments per document item

| Principal | Permission level | Source |
|---|---|---|
| Service account | Full Control | Always |
| Initiator (by email) | Read | Always |
| Org unit group (`orgUnitGroupId`) | Read | Always — from App Config |
| Additional users (`additionalUserEmails`) | Read | Passed by caller |
| Additional groups (`additionalGroupIds`) | Read | Passed by caller |

Permission levels reference existing SharePoint site permission level definitions.
The flow does not create new permission levels.

### Permission rules from App Config

The caller is responsible for reading `orgUnitGroupId` and any document-type-specific
permission overrides from App Config before calling this flow.
This flow falls back to a default group from App Config (UNKNOWN key) if `orgUnitGroupId`
is empty.

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varReadRoleDefId` (Integer) = 0

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- EventCategory: `Permission`
- Severity: `Info`
- Status: `Started`
- DocumentItemId: input `sviPredmetiItemId`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje dodele prava na dokument."

**Step 3 — Resolve Read role definition ID**
Action: SharePoint HTTP GET
URI: `<SharePointSite>/_api/web/roledefinitions?$filter=Name eq 'Read'&$select=Id`
Store `varReadRoleDefId` from the response.

This lookup is performed once per flow run and reused for all principal assignments.

**Step 4 — Resolve org unit group (fallback)**
If input `orgUnitGroupId` is empty:
- Read default group from App Config (UNKNOWN key for default org unit group).
- If still empty: log warning, continue without org unit group.

**Step 5 — Break permission inheritance on Svi predmeti item**
Action: SharePoint HTTP POST
URI: `<SharePointSite>/_api/web/lists/getByTitle('<SviPredmetiList>')/items(<sviPredmetiItemId>)/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)`

`copyRoleAssignments=false` — start with empty explicit permissions after break.

**Step 6 — Grant initiator Read on Svi predmeti item**
Action: SharePoint HTTP POST — ensureuser for `initiatorEmail` → get user ID.
Action: SharePoint HTTP POST — addroleassignment with `varReadRoleDefId`.
URI pattern: `/_api/web/lists/getByTitle('<SviPredmetiList>')/items(<sviPredmetiItemId>)/roleassignments/addroleassignment(principalid=<userId>,roledefid=<varReadRoleDefId>)`

**Step 7 — Grant org unit group Read on Svi predmeti item**
Condition: `orgUnitGroupId` is not empty.
Resolve group's SharePoint principal ID (see Open items — Entra group resolution method UNKNOWN).
Assign Read role.

**Step 8 — Grant additional users Read on Svi predmeti item**
Apply to each item in `additionalUserEmails` (if not empty):
- ensureuser → get user ID → addroleassignment Read.

**Step 9 — Grant additional groups Read on Svi predmeti item**
Apply to each item in `additionalGroupIds` (if not empty):
- Resolve group principal → addroleassignment Read.

**Step 10 — Apply same permissions to document library files**
Apply to each ID in `documentLibraryFileIds` (if not empty):
- Break inheritance on file item (same pattern, targeting `docDokumenti` library).
- Grant initiator Read.
- Grant org unit group Read (if set).
- Grant additional users Read.
- Grant additional groups Read.

**Step 11 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: input `sviPredmetiItemId`
- CorrelationId: `varCorrelationId`
- Message: "Prava su uspešno dodeljena."

**Step 12 — Return success response**

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `PermissionAssignment`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `PERMISSION_ASSIGNMENT_FAILED`
- CorrelationId: `varCorrelationId`

Return failure response.

## Caller responsibility

Callers must:
1. Look up the correct `orgUnitGroupId` from App Config before calling.
2. Pass the correct file IDs (if file-level permissions are needed).
3. Treat permission failure as non-fatal at document creation time.
4. At approval step transition (`ProcessApprovalResponse`): pass the new step's approver email or group ID in `additionalUserEmails` / `additionalGroupIds`.

## Error codes

| Code | Meaning |
|---|---|
| PERMISSION_ASSIGNMENT_FAILED | General failure |
| PERMISSION_BREAK_FAILED | Failed to break role inheritance |
| PERMISSION_USER_RESOLVE_FAILED | Could not resolve user via ensureuser |

## Open items

| Item | Status |
|---|---|
| App Config key for default org unit group | UNKNOWN |
| App Config key for document-type-specific permission overrides | UNKNOWN |
| Entra security group resolution to SharePoint principal (ensureuser vs group endpoint) | UNKNOWN — depends on site configuration |
| Whether file-level permissions are always applied or configurable via App Config | UNKNOWN — assume always for now |
| Permission retention policy for previous approvers after step advance | UNKNOWN — App Config key |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Standard document, single org unit group | Inheritance broken, initiator + group get Read |
| No org unit group configured | Inheritance broken, initiator gets Read, warning logged |
| Additional approver passed | Approver gets Read on item |
| Multiple files in documentLibraryFileIds | All files get same permissions |
| SharePoint permission API returns error | Failure logged, non-fatal to parent flow |
