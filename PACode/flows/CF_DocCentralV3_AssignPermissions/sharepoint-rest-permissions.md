# SharePoint REST API permission operations: CF_DocCentralV3_AssignPermissions

## Purpose

Reference guide for every SharePoint REST API call used to assign permissions.
All calls are made via the **SharePoint — Send an HTTP Request** action in Power Automate.
All calls use the `CR_DocCentralV3_SharePoint` connection reference.

---

## Request digest — required for all POST operations

SharePoint REST API write operations (POST, DELETE, PATCH/MERGE) require a form digest token
in the `X-RequestDigest` header. GET operations do not require it.

### How to obtain the request digest

Add **SharePoint — Send an HTTP Request** at the start of the Try scope (before any POST):

| Field | Value |
|---|---|
| Method | POST |
| Uri | `_api/contextinfo` |
| Headers | `Accept: application/json;odata=nometadata` |
| Body | (empty) |

Rename action: `Get_RequestDigest`

Extract the digest value:
```
outputs('Get_RequestDigest')?['body']?['FormDigestValue']
```

Use this value in the `X-RequestDigest` header of all subsequent POST actions.
The digest is valid for approximately 30 minutes. One digest per flow run is sufficient.

---

## 1 — Resolve Read role definition ID

**Purpose:** Get the integer ID of the SharePoint "Read" permission level.
Role definition IDs are site-specific — never hardcode them.

| Field | Value |
|---|---|
| Method | GET |
| Uri | `_api/web/roledefinitions?$filter=Name eq 'Read'&$select=Id` |
| Headers | `Accept: application/json;odata=nometadata` |

Rename action: `Get_ReadRoleDefinition`

Extract ID:
```
int(first(outputs('Get_ReadRoleDefinition')?['body']?['value'])?['Id'])
```

Store in `varReadRoleDefId`.

---

## 2 — Resolve user principal ID (ensureuser)

**Purpose:** Convert a user's email address or login name to a SharePoint principal ID
that can be used in role assignment calls.

| Field | Value |
|---|---|
| Method | POST |
| Uri | `_api/web/ensureuser` |
| Headers | `Accept: application/json;odata=nometadata`, `Content-Type: application/json;odata=nometadata`, `X-RequestDigest: <digest>` |
| Body | `{"logonName": "i:0#.f|membership|<userEmail>"}` |

**Login name format for SharePoint Online (claims-based):**
```
i:0#.f|membership|korisnik@organizacija.rs
```

If the plain email format (`korisnik@organizacija.rs`) does not work with ensureuser,
try the full claims format above. The format depends on the SharePoint tenant's authentication configuration.

Extract user ID:
```
int(outputs('EnsureUser_<Name>')?['body']?['Id'])
```

Add Condition after ensureuser: if status code is not 200 or body Id is 0 →
log Warning PERMISSION_USER_RESOLVE_FAILED, skip this user.

---

## 3 — Resolve Entra group to SharePoint principal

**Status: UNKNOWN — must be confirmed for the specific SharePoint Online tenant.**

Entra security groups are not the same as SharePoint site groups.
The method to resolve an Entra group object ID to a SharePoint principal ID depends on
how the tenant is configured. Two patterns are in common use:

### Pattern A — ensureuser with group claims format

Works for Entra security groups synced to SharePoint. Try:
```
Body: {"logonName": "c:0t.c|tenant|<entraGroupObjectId>"}
```
or
```
Body: {"logonName": "c:0o.c|federateddirectoryclaimprovider|<entraGroupObjectId>"}
```

The correct claims prefix varies by tenant. Test both against the actual SharePoint site.

### Pattern B — sitegroups endpoint (SharePoint groups only)

For SharePoint site-local groups (not Entra groups), use:
```
GET _api/web/sitegroups/getbyname('<groupName>')
```
This returns the group's SharePoint principal ID. Not applicable for Entra groups.

### Pattern C — Office365Groups connector

If using M365 Groups (not security groups), the `CR_DocCentralV3_Office365Groups` connection
reference provides list-members and group resolution actions without REST calls.
However, to get a SharePoint principal ID for role assignment, you still need to resolve
the group to a SharePoint-level principal.

### Decision required before building

Before building this flow, confirm:
1. Are the org unit groups Entra security groups, M365 groups, or SharePoint site groups?
2. What claims format does the tenant use for ensureuser?
3. Test Pattern A against the actual site with a known group object ID.

Until confirmed, mark all Entra group resolution steps as UNKNOWN in the build.

---

## 4 — Break role inheritance

**Purpose:** Remove inherited permissions from a list item and start with an empty permission set.

### On Svi predmeti item

| Field | Value |
|---|---|
| Method | POST |
| Uri | `_api/web/lists/getbytitle('<ListName>')/items(<itemId>)/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)` |
| Headers | `Accept: application/json;odata=nometadata`, `Content-Type: application/json;odata=nometadata`, `X-RequestDigest: <digest>` |
| Body | (empty) |

Expected status: `200` or `204`.

Replace `<ListName>` using EV expression inside concat:
```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'),
  ''')/items(',
  string(variables('varSviPredmetiItemId')),
  ')/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)'
)
```

Switch the Uri field to Expression mode and paste this.

### On Dokumenti library file

Same pattern, using `EV_DocCentralV3_docDokumenti` and the file item ID:
```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_docDokumenti'),
  ''')/items(',
  string(items('Apply_to_each_Files')),
  ')/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)'
)
```

---

## 5 — Add role assignment (grant permission)

**Purpose:** Grant a specific permission level to a principal on a specific item.

### On Svi predmeti item

| Field | Value |
|---|---|
| Method | POST |
| Uri | `_api/web/lists/getbytitle('<ListName>')/items(<itemId>)/roleassignments/addroleassignment(principalid=<principalId>,roledefid=<roleDefId>)` |
| Headers | `Accept: application/json;odata=nometadata`, `Content-Type: application/json;odata=nometadata`, `X-RequestDigest: <digest>` |
| Body | (empty) |

Expected status: `200` or `204`.

Expression for Uri (Svi predmeti, initiator user ID):
```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'),
  ''')/items(',
  string(variables('varSviPredmetiItemId')),
  ')/roleassignments/addroleassignment(principalid=',
  string(<principalId>),
  ',roledefid=',
  string(variables('varReadRoleDefId')),
  ')'
)
```

Replace `<principalId>` with the resolved integer ID from ensureuser or group resolution.

### On Dokumenti library file

Same pattern, using `EV_DocCentralV3_docDokumenti` and the file item ID from the loop.

---

## 6 — Remove role assignment (revoke permission)

**Purpose:** Remove a specific principal's permission from an item.
Used when revoking a previous approver's access at step advance (if retention policy requires removal).

| Field | Value |
|---|---|
| Method | POST |
| Uri | `_api/web/lists/getbytitle('<ListName>')/items(<itemId>)/roleassignments/getbyprincipalid(<principalId>)/DeleteObject()` |
| Headers | `Accept: application/json;odata=nometadata`, `Content-Type: application/json;odata=nometadata`, `X-RequestDigest: <digest>` |
| Body | (empty) |

**Note:** This operation is not used in v1 (previous approvers retain access by default).
Document here for reference when the retention policy is decided.

---

## 7 — Check current role assignments (diagnostic)

**Purpose:** Read existing role assignments on an item. Useful for testing and validation.

| Field | Value |
|---|---|
| Method | GET |
| Uri | `_api/web/lists/getbytitle('<ListName>')/items(<itemId>)/roleassignments?$expand=Member,RoleDefinitionBindings&$select=Member/LoginName,Member/Id,RoleDefinitionBindings/Name` |
| Headers | `Accept: application/json;odata=nometadata` |

Use this in a test flow to verify permissions were applied correctly after each test case.

---

## Common HTTP status codes

| Status | Meaning in this context |
|---|---|
| 200 | OK — operation succeeded |
| 204 | No Content — operation succeeded (common for POST operations that return no body) |
| 400 | Bad Request — malformed URI, invalid principal ID, or missing request digest |
| 403 | Forbidden — service account lacks permission to manage this item's permissions |
| 404 | Not Found — list, item, or role definition does not exist |
| 409 | Conflict — role assignment already exists (safe to ignore for idempotent re-application) |

---

## Power Automate — Send an HTTP Request action notes

- The action is named **SharePoint — Send an HTTP Request** in the connector.
- It is a standard (non-premium) action.
- Set Method field to `POST` for write operations — even operations that feel like they should be PATCH (SharePoint REST uses POST + X-HTTP-Method: MERGE for updates, but role assignments use pure POST).
- The `X-RequestDigest` header must be included in all write (POST) operations.
- Site Address field: use the EV expression `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`.
- Uri field: use Expression mode (not dynamic content) when embedding EV references inside the URI string via concat.

---

## EV reference inside URI — recap

All list names in URIs must be referenced via EV expressions, not hardcoded.

Pattern for concat in Uri field (Expression mode):
```
concat('_api/web/lists/getbytitle(''', parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'), ''')/items(', string(variables('varSviPredmetiItemId')), ')/...')
```

The double single-quotes (`''`) inside the concat are how you escape a literal single quote
within a Power Automate string expression. The outer single quotes belong to the OData query syntax.
