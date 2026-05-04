# Permission model: CF_DocCentralV3_AssignPermissions

## Overview

DocCentralV3 uses a **break-inheritance + explicit grant** model for document-level access control.
SharePoint list-level and site-level permissions remain set to Read for regular users.
Per-document access is controlled by breaking role inheritance on the Svi predmeti item
and its associated files in the Dokumenti library, then assigning explicit Read grants.

---

## Site-level baseline

| Principal | Site permission level |
|---|---|
| All document users | Read (site and list level) |
| Service account | Full Control |
| DocCentralV3 administrators | Full Control |

These permissions apply to lists and libraries that have not had inheritance broken.
They are configured at site deployment time — not managed by this flow.

---

## Per-document permission model

When this flow runs, it:
1. Breaks role inheritance on the Svi predmeti item (and optionally on each Dokumenti file).
2. Assigns explicit permissions to a set of configured principals.
3. The service account retains Full Control implicitly because it is the owner of the connection reference and has full control at the site level. After breaking inheritance with `copyRoleAssignments=false`, the service account's site-level Full Control still applies. Verify this post-break for the specific SharePoint site configuration.

### Principals granted access per document

| Principal | Permission level | Source | Condition |
|---|---|---|---|
| Service account | Full Control | Site-level (not re-granted by this flow) | Always |
| Initiator (by email) | Read | `initiatorEmail` input | Always |
| Org unit group | Read | `orgUnitGroupId` input or App Config fallback | If group is resolved |
| Additional users | Read | `additionalUserEmails` input | If array is non-empty |
| Additional groups | Read | `additionalGroupIds` input | If array is non-empty |

### What this flow does NOT grant

- Write or Edit permission to any user (only service account has write access).
- Contribute permission level — not used in DocCentralV3.
- Access to the site or list level — only item-level explicit grants are set.
- Access based on document content or metadata — the caller must derive the correct principals before calling.

---

## Break inheritance settings

`breakroleinheritance(copyRoleAssignments=false, clearSubscopes=true)`

- `copyRoleAssignments=false`: After break, the item starts with **no explicit permissions**. All grants are applied from scratch by this flow. This avoids accumulating legacy permissions from list-level defaults.
- `clearSubscopes=true`: Clears any sub-item inheritance breaks (relevant for document library folders). Safe to use on list items that have no sub-items.

---

## Re-application behaviour

Each call to this flow breaks inheritance and re-applies all permissions from the inputs.
This is intentional — it allows the same flow to be called at:
- Document creation (initial grant: initiator + org unit group)
- SendForApproval (add step 1 approver or approver group)
- ProcessApprovalResponse step advance (add step 2 approver, remove step 1 approver if not retained)

**Important:** Because each call starts fresh (copyRoleAssignments=false), callers at approval
step transitions must pass the **full set of principals who should have access**, not only the
newly added approver. If the previous approver should retain access, pass their email in
`additionalUserEmails`. If they should lose access, do not include them.

---

## Permission retention policy for previous approvers

**Status: UNKNOWN — requires decision.**

When an approval step advances from step 1 to step 2:
- Option A: Step 1 approver loses access (not passed in additionalUserEmails for step 2 call).
- Option B: Step 1 approver retains access (passed in additionalUserEmails for step 2 call).

The policy choice is a client business decision, likely configurable via ProcesConfig or App Config.
The App Config key for this setting is UNKNOWN.

Until this is resolved:
- **Default for v1:** Previous approvers retain access (callers accumulate additionalUserEmails across steps).
- Document this default in the CLAUDE.md or implementation notes once confirmed.

---

## Org unit group resolution

Group IDs must be Entra object IDs (GUIDs), not display names. Example: `00000000-0000-0000-0000-000000000001`.

The caller (e.g. CF_DocCentralV3_CreateDocument) is responsible for reading the org unit group ID
from App Config before calling this flow. This flow accepts the ID as input and does not perform
App Config lookups for document-type-specific mappings — it only performs the fallback lookup
for the default org unit group when `orgUnitGroupId` is empty.

**App Config keys (all UNKNOWN):**

| Config entry | Purpose | Key name |
|---|---|---|
| Default org unit group | Fallback when orgUnitGroupId not provided by caller | UNKNOWN |
| Document-type permission overrides | Optional per-type access rules | UNKNOWN |
| Previous approver retention policy | Whether prior step approvers keep access | UNKNOWN |

---

## Non-fatal permission failure policy

Permission failure during document creation is **non-fatal**:

| Failure type | Severity | Action |
|---|---|---|
| Entire flow fails (Catch scope) | Error | Log PERMISSION_ASSIGNMENT_FAILED. Parent flow continues. Admin alert via AuditLog. |
| Break inheritance fails | Error | PERMISSION_BREAK_FAILED. Terminate to Catch. Non-fatal to parent. |
| Single user ensureuser fails | Warning | Log PERMISSION_USER_RESOLVE_FAILED. Skip that user. Continue loop. |
| Org unit group not found in App Config | Warning | Log warning. Continue without group grant. |

The rationale: the document exists after creation. Stopping or rolling back the document
creation due to a permission error would be more disruptive than a document with temporarily
incomplete permissions. The AuditLog Critical entry notifies administrators to repair.

Permission failure at approval step transitions follows the same non-fatal policy.

---

## What administrators must do when PERMISSION_ASSIGNMENT_FAILED is logged

1. Find the document item in Svi predmeti.
2. Manually re-run permissions: call this flow with the correct inputs from a test flow, or
   manually set item permissions in SharePoint.
3. Verify the document is accessible to the correct users.
4. Mark the AuditLog entry as resolved (or add a Success entry manually for tracking).

---

## File-level permission considerations

Files in the Dokumenti library receive the same permission set as their parent Svi predmeti item.

- Each file has inheritance broken individually (not the entire library).
- The same principals (initiator, org unit group, additional users/groups) get Read on each file.
- If `documentLibraryFileIds` is empty, only the Svi predmeti item is permission-set.
  File access falls back to the library default (Read for all authenticated users at site level).
- File IDs must be SharePoint list item IDs (integer), not file names or unique IDs.

---

## Permission level used: Read

The SharePoint **Read** permission level is used for all non-service-account grants.
This is a standard SharePoint permission level present on all SharePoint Online sites by default.

Read allows:
- View items, documents, and their versions.
- View list pages and site pages.

Read does NOT allow:
- Creating, editing, or deleting items.
- Managing permissions.
- Approving or checking out documents.

This aligns with the DocCentralV3 security model: all writes go through Power Automate flows
running under the service account. Users are consumers, not editors, of SharePoint content.

---

## Entra group type considerations

SharePoint's permission system distinguishes between:
- **SharePoint groups** (site-local groups): granted directly via group ID.
- **Entra security groups** (M365): require the `ensureuser` endpoint or the `/_api/web/sitegroups` pattern.
- **Microsoft 365 groups (Teams)**: similar to security groups but may require different resolution.

The method to resolve an Entra group to a SharePoint principal ID is UNKNOWN.
See `sharepoint-rest-permissions.md` for the patterns to evaluate and confirm.
