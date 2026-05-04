# Implementation steps: CF_DocCentralV3_AssignPermissions

## Prerequisites

- CF_DocCentralV3_LogEvent is built and published in the DocCentralV3 solution.
- Svi predmeti SharePoint list exists.
- Dokumenti document library exists.
- App Config list exists with a default org unit group entry (key UNKNOWN).
- AuditLog SharePoint list exists.
- Connection references connected:
  - CR_DocCentralV3_SharePoint
  - CR_DocCentralV3_Office365Groups (used for Entra group resolution if applicable — see `sharepoint-rest-permissions.md`)
- Environment variables registered:
  EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_docDokumenti,
  EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog.
- The service account used by CR_DocCentralV3_SharePoint has Full Control on Svi predmeti and Dokumenti.
- The SharePoint site has a permission level named **Read** defined (standard — present on all SharePoint sites by default).

---

## Step 1 — Create the flow

1. Open the DocCentralV3 solution.
2. New → Cloud flow → Instant cloud flow.
3. Name: `CF_DocCentralV3_AssignPermissions`.
4. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Replace trigger with Power Apps (V2)

1. Delete the placeholder trigger.
2. Add trigger: **Power Apps (V2)**.
3. Add inputs:

| Input name | Type |
|---|---|
| sviPredmetiItemId | Number |
| documentLibraryFileIds | Text |
| initiatorEmail | Text |
| documentType | Text |
| orgUnitGroupId | Text |
| additionalUserEmails | Text |
| additionalGroupIds | Text |
| correlationId | Text |

**Array inputs (documentLibraryFileIds, additionalUserEmails, additionalGroupIds):**
Power Apps V2 trigger does not support Array type inputs. Pass arrays as JSON strings
(Text type) and parse them inside the flow using `json()`. Example:
- Caller passes: `"[1, 2, 3]"` as documentLibraryFileIds
- Flow parses: `json(triggerBody()?['text_documentLibraryFileIds'])`

If empty (no files / no additional users), callers pass `"[]"`.

Internal references:
- `triggerBody()?['decimal_sviPredmetiItemId']`
- `triggerBody()?['text_documentLibraryFileIds']`
- `triggerBody()?['text_initiatorEmail']`
- `triggerBody()?['text_documentType']`
- `triggerBody()?['text_orgUnitGroupId']`
- `triggerBody()?['text_additionalUserEmails']`
- `triggerBody()?['text_additionalGroupIds']`
- `triggerBody()?['text_correlationId']`

---

## Step 3 — Initialize variables

Add **Initialize variable** actions before the Try scope:

| Variable name | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varReadRoleDefId | Integer | `0` |
| varOrgUnitGroupId | String | `triggerBody()?['text_orgUnitGroupId']` |
| varSviPredmetiItemId | Integer | `int(triggerBody()?['decimal_sviPredmetiItemId'])` |
| varFileIds | Array | `json(triggerBody()?['text_documentLibraryFileIds'])` |
| varAdditionalUserEmails | Array | `json(triggerBody()?['text_additionalUserEmails'])` |
| varAdditionalGroupIds | Array | `json(triggerBody()?['text_additionalGroupIds'])` |

Initialize varCorrelationId first.

---

## Step 4 — Log Started

Add **Run a Child Flow** — CF_DocCentralV3_LogEvent:

| Parameter | Value |
|---|---|
| eventType | `PermissionAssignment` |
| eventCategory | `Permission` |
| severity | `Info` |
| status | `Started` |
| correlationId | `variables('varCorrelationId')` |
| flowName | `CF_DocCentralV3_AssignPermissions` |
| flowRunId | `workflow().run.name` |
| source | `CloudFlow` |
| userEmail | `triggerBody()?['text_initiatorEmail']` |
| documentItemId | `variables('varSviPredmetiItemId')` |
| message | `Pokretanje dodele prava na dokument.` |

---

## Step 5 — Try scope

Add **Scope** action. Name it `Try`. All permission logic goes inside.

### 5a — Resolve the Read role definition ID

Add **SharePoint — Send an HTTP Request**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Method: GET
- Uri: `_api/web/roledefinitions?$filter=Name eq 'Read'&$select=Id`
- Headers:
  ```
  Accept: application/json;odata=nometadata
  ```

Rename action: `Get_ReadRoleDefinition`

Set varReadRoleDefId:
```
int(first(outputs('Get_ReadRoleDefinition')?['body']?['value'])?['Id'])
```

If the response body is empty (Read role not found), add a Condition and Terminate Failed
to trigger the Catch scope.

### 5b — Resolve org unit group (fallback to App Config default)

Add **Condition**:
- Left: `empty(variables('varOrgUnitGroupId'))`
- Operator: is equal to
- Right: `true`

Yes branch (orgUnitGroupId is empty — read default from App Config):
1. Add **SharePoint — Get items** on App Config:
   - Filter: UNKNOWN — the key identifying the default org unit group. See `permission-model.md`.
   - Top: 1
2. Condition: if result is empty → log Warning (no org unit group configured), leave varOrgUnitGroupId empty, continue.
3. If result found: Set varOrgUnitGroupId from App Config value (UNKNOWN column name).

No branch (orgUnitGroupId provided by caller): continue.

### 5c — Break inheritance on Svi predmeti item

Add **SharePoint — Send an HTTP Request**:
- Method: POST
- Uri:
  ```
  _api/web/lists/getbytitle('@{parameters(''gpdoccen_EV_DocCentralV3_lstSviPredmeti'')}'/items(@{variables('varSviPredmetiItemId')})/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)
  ```
- Headers:
  ```
  Accept: application/json;odata=nometadata
  Content-Type: application/json;odata=nometadata
  X-RequestDigest: @{outputs('Get_RequestDigest')?['body']?['d']?['GetContextWebInformation']?['FormDigestValue']}
  ```

**Request digest:** SharePoint REST POST operations require a form digest value.
See `sharepoint-rest-permissions.md` for how to obtain and pass the request digest.
Rename action: `Break_Inheritance_SviPredmeti`

Add Condition on response status code:
- If not 200 or 204: set errorCode `PERMISSION_BREAK_FAILED`, Terminate Failed.

### 5d — Grant initiator Read on Svi predmeti item

Add **SharePoint — Send an HTTP Request** — ensureuser:
- Method: POST
- Uri: `_api/web/ensureuser`
- Body: `{"logonName": "@{triggerBody()?['text_initiatorEmail']}"}`

Rename: `EnsureUser_Initiator`

Extract user ID:
```
int(outputs('EnsureUser_Initiator')?['body']?['Id'])
```

Add Condition: if user ID is 0 or action failed → log Warning with errorCode `PERMISSION_USER_RESOLVE_FAILED`, continue (non-fatal).

Add **SharePoint — Send an HTTP Request** — addroleassignment:
- Method: POST
- Uri:
  ```
  _api/web/lists/getbytitle('@{parameters(''gpdoccen_EV_DocCentralV3_lstSviPredmeti'')}'/items(@{variables('varSviPredmetiItemId')})/roleassignments/addroleassignment(principalid=@{<initiatorUserId>},roledefid=@{variables('varReadRoleDefId')})
  ```

Rename: `Grant_Read_Initiator_SviPredmeti`

### 5e — Grant org unit group Read on Svi predmeti item

Add **Condition**:
- Left: `empty(variables('varOrgUnitGroupId'))`
- Operator: is equal to
- Right: `false`

Yes branch (group is set):
1. Resolve Entra group to SharePoint principal — see `sharepoint-rest-permissions.md` for the ensureuser / sitegroups pattern. Method is UNKNOWN depending on whether it is an Entra security group or an M365 group.
2. Add addroleassignment for the resolved principal ID with varReadRoleDefId.

Rename relevant actions: `Resolve_OrgUnitGroup`, `Grant_Read_OrgUnitGroup_SviPredmeti`

### 5f — Grant additional users Read on Svi predmeti item

Add **Apply to each** over `variables('varAdditionalUserEmails')`:
- Current item reference: `items('Apply_to_each_AdditionalUsers')`
- Inside: ensureuser → extract user ID → addroleassignment Read on Svi predmeti item.

Rename: `Apply_to_each_AdditionalUsers`

Skip items where ensureuser fails — log Warning per failed user, continue loop.

### 5g — Grant additional groups Read on Svi predmeti item

Add **Apply to each** over `variables('varAdditionalGroupIds')`:
- Inside: resolve group principal → addroleassignment Read on Svi predmeti item.

Rename: `Apply_to_each_AdditionalGroups`

### 5h — Apply permissions to document library files

Add **Condition**:
- Left: `length(variables('varFileIds'))`
- Operator: greater than
- Right: `0`

Yes branch:
Add **Apply to each** over `variables('varFileIds')`. For each file ID:

1. Break inheritance on file:
   - Uri: `_api/web/lists/getbytitle('@{parameters(''gpdoccen_EV_DocCentralV3_docDokumenti'')}'/items(@{items('Apply_to_each_Files')})/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)`
2. Grant initiator Read on file (same ensureuser pattern — reuse initiator user ID from Step 5d if still in scope, otherwise re-resolve).
3. Grant org unit group Read on file (if varOrgUnitGroupId is not empty).
4. Grant additional users Read on file (nested Apply to each).
5. Grant additional groups Read on file (nested Apply to each).

Rename outer loop: `Apply_to_each_Files`

**Note:** Nested Apply to each loops are supported in Power Automate but can be slow for large arrays. Keep file arrays small. If performance is an issue in production, consider batching via HTTP requests.

### 5i — Log Success

Add **Run a Child Flow** — CF_DocCentralV3_LogEvent:
- eventType: `PermissionAssignment` / severity: `Info` / status: `Success`
- message: `Prava su uspešno dodeljena.`

### 5j — Respond success

Add **Respond to a PowerApp or flow** — success outputs.

---

## Step 6 — Catch scope

Add **Scope** after Try. Name it `Catch`.
Configure Run after: uncheck `is successful`, check `has failed`, check `has timed out`.

Inside Catch:
1. Call CF_DocCentralV3_LogEvent:
   - eventType: `PermissionAssignment` / severity: `Error` / status: `Failed`
   - errorCode: `PERMISSION_ASSIGNMENT_FAILED`
   - errorMessage: `result('Try')?[0]?['error']?['message']`
2. Add **Respond to a PowerApp or flow** — failure outputs.

---

## Step 7 — Post-scope routing

Add **Condition** after both scopes:
- Left: `result('Try')?[0]?['status']` equals `Succeeded`

Yes: flow already responded in Step 5j — no action needed.
No: Catch scope already responded.

(This step is optional if all Respond actions are already placed in Try and Catch. Include it if you want a guaranteed final Respond as a safety net.)

---

## Step 8 — Respond outputs

**Success:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `true` |
| message | Text | `Prava su uspešno dodeljena.` |
| correlationId | Text | `variables('varCorrelationId')` |
| errorCode | Text | `''` |

**Failure:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `false` |
| message | Text | `Greška pri dodeli prava.` |
| correlationId | Text | `variables('varCorrelationId')` |
| errorCode | Text | `PERMISSION_ASSIGNMENT_FAILED` |

---

## Step 9 — Save and verify

1. Save frequently — this flow has many HTTP actions and is complex to rebuild.
2. Verify flow is inside DocCentralV3 solution.
3. Run TC-001 through TC-004 before integrating with CF_DocCentralV3_CreateDocument.
4. Description: `"v1.0 — initial implementation. CF_DocCentralV3_AssignPermissions. DocCentralV3 solution."`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_AssignPermissions`
- [ ] Trigger is Power Apps (V2) with 8 inputs (array inputs as Text/JSON strings)
- [ ] All 7 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Log Started called before Try scope
- [ ] Read role definition ID resolved via REST GET (not hardcoded)
- [ ] Org unit group fallback: reads App Config default when orgUnitGroupId is empty
- [ ] Warning logged (non-fatal) when no org unit group is available
- [ ] Inheritance broken on Svi predmeti item (copyRoleAssignments=false)
- [ ] Break failure triggers Terminate Failed → Catch scope
- [ ] Initiator gets Read via ensureuser + addroleassignment
- [ ] Org unit group gets Read (if resolved)
- [ ] Additional users loop: each user gets Read (failed ensureuser is Warning, not fatal)
- [ ] Additional groups loop: each group gets Read
- [ ] File permissions: inheritance broken and same grants applied per file
- [ ] Service account retains Full Control implicitly (it is the connection owner — verify post-break)
- [ ] Log Success called after all assignments
- [ ] Catch scope logs Error, returns failure response
- [ ] Catch scope does NOT call CF_DocCentralV3_AssignPermissions
- [ ] All 4 output parameters in both Respond actions
- [ ] All SharePoint HTTP actions use CR_DocCentralV3_SharePoint
- [ ] All EV references use @parameters() syntax
- [ ] Request digest obtained and passed to all POST actions (see sharepoint-rest-permissions.md)
- [ ] All UNKNOWN items resolved before building
- [ ] Flow is inside DocCentralV3 solution
