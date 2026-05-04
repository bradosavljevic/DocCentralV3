# Power Automate build notes: CF_DocCentralV3_AssignPermissions

## Purpose of this document

Practical notes and gotchas for building this flow in the Power Automate maker portal.
Read alongside `implementation-steps.md`, `permission-model.md`, and `sharepoint-rest-permissions.md`.

---

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_AssignPermissions |
| Solution | DocCentralV3 |
| Trigger | Power Apps (V2) — child flow pattern |
| Called by | CF_DocCentralV3_CreateDocument, CF_DocCentralV3_SendForApproval, CF_DocCentralV3_ProcessApprovalResponse |
| Connection references | CR_DocCentralV3_SharePoint, CR_DocCentralV3_Office365Groups |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_docDokumenti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog |

---

## Array inputs — Power Apps V2 limitation

The Power Apps V2 trigger does not support native Array input types.
All three array inputs must be declared as **Text** and serialized as JSON strings by callers.

| Input | Declared type | Example value from caller |
|---|---|---|
| documentLibraryFileIds | Text | `"[101, 102]"` or `"[]"` |
| additionalUserEmails | Text | `"[\"user@org.rs\"]"` or `"[]"` |
| additionalGroupIds | Text | `"[\"00000000-...\"]"` or `"[]"` |

Inside the flow, parse these immediately after trigger using Initialize variable:
```
varFileIds = json(triggerBody()?['text_documentLibraryFileIds'])
```

If the caller passes `null` instead of `"[]"`, `json(null)` will fail.
Add a Condition before parsing: if the trigger value is empty, default to `"[]"`:
```
if(empty(triggerBody()?['text_documentLibraryFileIds']), json('[]'), json(triggerBody()?['text_documentLibraryFileIds']))
```

---

## Request digest — obtain once, use everywhere

Every POST to the SharePoint REST API requires a form digest in `X-RequestDigest`.
Get it once at the start of the Try scope:

1. Add **SharePoint — Send an HTTP Request** — POST to `_api/contextinfo`.
2. Rename: `Get_RequestDigest`.
3. Extract: `outputs('Get_RequestDigest')?['body']?['FormDigestValue']`
4. Store in a variable `varRequestDigest` (String type).

Use `variables('varRequestDigest')` in the `X-RequestDigest` header of every subsequent POST.

**Do not request a new digest per action** — one per flow run is sufficient (valid ~30 min).

---

## EV reference inside URI — expression mode required

Embedding environment variable values inside the URI field of Send an HTTP Request requires
switching the Uri field to **Expression mode** (not dynamic content mode).

Use `concat()` to build the URI string. Single quotes inside list names must be doubled:

```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'),
  ''')/items(',
  string(variables('varSviPredmetiItemId')),
  ')/breakroleinheritance(copyRoleAssignments=false,clearSubscopes=true)'
)
```

The outer single-quotes are the OData query syntax for string literals.
The doubled `''` inside concat is the Power Automate escape for a literal single-quote inside an expression string.

---

## ensureuser login name format

For SharePoint Online, the login name for `ensureuser` must use the claims format:
```
i:0#.f|membership|korisnik@organizacija.rs
```

If plain email fails, always try the claims format. The correct format depends on
the tenant authentication configuration. Test with a known user before building the loop.

In the JSON body:
```json
{"logonName": "i:0#.f|membership|@{triggerBody()?['text_initiatorEmail']}"}
```

Inside Apply to each (additional users):
```json
{"logonName": "i:0#.f|membership|@{items('Apply_to_each_AdditionalUsers')}"}
```

---

## Apply to each — concurrency setting

By default, Apply to each loops run **sequentially** (one iteration at a time).
For small arrays (1–3 files or users) this is fine.

For larger arrays, consider enabling concurrency (max 10 parallel iterations).
Power Automate: Apply to each → Settings → Concurrency Control → On → Degree of Parallelism = 5.

**Warning:** Do not use high concurrency for permission operations — SharePoint REST API
may throttle concurrent permission writes with 429 responses.
Recommended: keep concurrency at 1 (sequential) for all permission loops unless performance
testing shows it is necessary.

---

## Handling 204 vs 200 responses from permission endpoints

Many SharePoint REST permission operations return **204 No Content** (success, empty body)
rather than 200. Do not check for response body content after permission POST calls.

Check only the status code:
- 200 or 204 → success
- Anything else → failure

Condition expression:
```
or(equals(outputs('Break_Inheritance_SviPredmeti')?['statusCode'], 200), equals(outputs('Break_Inheritance_SviPredmeti')?['statusCode'], 204))
```

---

## Nested Apply to each — file permission loop

The file permission loop (Apply to each over varFileIds) contains:
- One break inheritance call
- One ensureuser call (or reuse of initiator user ID from outer scope)
- One or more role assignment calls

**Reusing initiator user ID:** If the initiator's user ID was captured in a variable (e.g. `varInitiatorUserId`) in Step 5d, it is accessible inside the nested loop without another ensureuser call. Store it:
```
varInitiatorUserId = int(outputs('EnsureUser_Initiator')?['body']?['Id'])
```

Similarly, store the org unit group's principal ID in `varOrgUnitPrincipalId` after resolution in Step 5e.
Reuse both inside the file loop to avoid repeated API calls.

---

## Action naming strategy

With many similar HTTP Request actions, name them clearly:

| Action purpose | Suggested name |
|---|---|
| Get request digest | `Get_RequestDigest` |
| Get Read role definition | `Get_ReadRoleDefinition` |
| Ensure initiator user | `EnsureUser_Initiator` |
| Resolve org unit group | `Resolve_OrgUnitGroup` |
| Break inheritance on Svi predmeti | `Break_Inheritance_SviPredmeti` |
| Grant Read to initiator (Svi predmeti) | `Grant_Read_Initiator_SviPredmeti` |
| Grant Read to org unit group (Svi predmeti) | `Grant_Read_OrgUnit_SviPredmeti` |
| Apply to each — additional users | `Apply_to_each_AdditionalUsers` |
| Apply to each — additional groups | `Apply_to_each_AdditionalGroups` |
| Apply to each — files | `Apply_to_each_Files` |
| Break inheritance on file (inside loop) | `Break_Inheritance_File` |

Using `outputs('<ActionName>')` in expressions requires the action's internal name.
Power Automate replaces spaces with underscores in internal names automatically.

---

## Service account permission retention after break inheritance

After calling `breakroleinheritance(copyRoleAssignments=false)`, the item starts with **no explicit grants**.
The service account's **site-level Full Control does still apply** because:
- The service account is the connection owner.
- Site-level permission groups (e.g. Owners) retain access even after item-level breaks.

**Verify this post-break for the actual SharePoint site.** If the service account loses access
after the break (because it is only in a list-level permission group, not the site Owners group),
you must explicitly re-grant Full Control using addroleassignment with the Full Control role def ID.

To get the Full Control role definition ID (if needed):
```
GET _api/web/roledefinitions?$filter=Name eq 'Full Control'&$select=Id
```

Store in `varFullControlRoleDefId` and add a grant step for the service account after break.

---

## CF_DocCentralV3_LogEvent calls in this flow

| When | eventType | severity | status | notes |
|---|---|---|---|---|
| Before Try scope | PermissionAssignment | Info | Started | Always called |
| After all grants succeed | PermissionAssignment | Info | Success | End of Try scope |
| No org unit group configured | PermissionAssignment | Warning | — | Non-fatal warning, flow continues |
| User ensureuser fails | PermissionAssignment | Warning | — | Non-fatal per user, loop continues |
| Catch scope | PermissionAssignment | Error | Failed | PERMISSION_ASSIGNMENT_FAILED |

**Critical:** The Catch scope must NEVER call CF_DocCentralV3_AssignPermissions.
Only CF_DocCentralV3_LogEvent is called from Catch.

---

## Environment variable references

| EV | Expression |
|---|---|
| EV_DocCentralV3_SharePointSite | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| EV_DocCentralV3_lstSviPredmeti | `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')` |
| EV_DocCentralV3_docDokumenti | `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')` |
| EV_DocCentralV3_lstAppConfig | `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')` |
| EV_DocCentralV3_lstAuditLog | `@parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')` |

---

## Known open items — must resolve before building

| Item | Where documented |
|---|---|
| Entra group resolution method (claims format or API pattern) | `sharepoint-rest-permissions.md` — Patterns A/B/C |
| App Config key for default org unit group | `permission-model.md` |
| App Config key for document-type permission overrides | `permission-model.md` |
| Previous approver retention policy (App Config key) | `permission-model.md` |
| Whether service account needs explicit re-grant after break | `power-automate-build-notes.md` — section above |

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_AssignPermissions`
- [ ] Trigger is Power Apps (V2) with 8 inputs (array inputs as Text)
- [ ] All 7 variables initialized before Try scope (including varRequestDigest if stored as variable)
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Array inputs parsed with json() into Array variables, null-safe
- [ ] Log Started called before Try scope
- [ ] Request digest obtained at start of Try scope (POST to _api/contextinfo)
- [ ] Read role definition ID fetched via REST GET (not hardcoded)
- [ ] Org unit group fallback reads App Config when input is empty
- [ ] Warning logged when org unit group not available (non-fatal)
- [ ] Break inheritance on Svi predmeti: copyRoleAssignments=false
- [ ] Break failure → Terminate Failed → Catch scope
- [ ] Initiator ensureuser + addroleassignment Read (Svi predmeti)
- [ ] Org unit group resolved and granted Read (Svi predmeti) when available
- [ ] Additional users loop: each user gets Read; failed ensureuser is Warning, not Fatal
- [ ] Additional groups loop: each group gets Read
- [ ] File loop: break + full permission set per file
- [ ] varInitiatorUserId and varOrgUnitPrincipalId reused inside file loop (no duplicate API calls)
- [ ] All POST actions include X-RequestDigest header
- [ ] Status code checked after break inheritance and role assignments
- [ ] Log Success after all assignments complete
- [ ] Catch scope: LogEvent Error, Respond failure (4 outputs)
- [ ] Catch scope does NOT call CF_DocCentralV3_AssignPermissions
- [ ] All URI expressions use concat + parameters() for EV references (no hardcoded list names)
- [ ] All SharePoint HTTP actions use CR_DocCentralV3_SharePoint
- [ ] All UNKNOWN items resolved before building
- [ ] Flow is inside DocCentralV3 solution
