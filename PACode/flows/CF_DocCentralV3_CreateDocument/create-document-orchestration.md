# Document creation orchestration: CF_DocCentralV3_CreateDocument

## Role in the solution

CF_DocCentralV3_CreateDocument is the central orchestration flow for document registration.
It is the only entry point through which a new document can be registered in DocCentralV3.
The Canvas App must not write directly to SharePoint for document creation.

This flow owns the complete document creation lifecycle:
- Input validation
- Registry number acquisition (generated or reserved)
- Svi predmeti item creation
- Main file upload
- Attachment upload
- Permission assignment
- Reserved number cleanup
- Structured response to Canvas App

---

## Full execution sequence

```
Canvas App (scrNoviPredmet)
│
└─ CALL CF_DocCentralV3_CreateDocument
   │
   ├─ Initialize variables
   ├─ Log Started (CF_DocCentralV3_LogEvent)
   │
   ├─ [TRY SCOPE]
   │  │
   │  ├─ Validate inputs (7 checks)
   │  ├─ Validate documentType vs App Config
   │  ├─ Validate active year / registry book
   │  │
   │  ├─ IF useReservedNumber = false:
   │  │   └─ CALL CF_DocCentralV3_GenerateRegistryNumber
   │  │       → varDelovodniBroj, varFilingDate = today
   │  │
   │  ├─ IF useReservedNumber = true:
   │  │   └─ CALL CF_DocCentralV3_UseReservedNumber
   │  │       → varDelovodniBroj, varFilingDate = input filingDate
   │  │
   │  ├─ Build varSafeDelovodniBroj (sanitize for file names)
   │  │
   │  ├─ Create Svi predmeti item
   │  │   → varSviPredmetiItemId
   │  │
   │  ├─ Generate main file system name (with guid)
   │  ├─ Create main file in Dokumenti root
   │  │   → varMainFileId → append to varCreatedFileIds
   │  ├─ Update main file metadata
   │  │
   │  ├─ [IF attachments exist]
   │  │   └─ For each attachment:
   │  │       ├─ Generate attachment system name (_PRILOG_guid_)
   │  │       ├─ Create attachment file in Dokumenti root
   │  │       │   → append ID to varCreatedFileIds
   │  │       └─ Update attachment metadata (IsPrilog=true, ParentDocumentId)
   │  │
   │  ├─ CALL CF_DocCentralV3_AssignPermissions
   │  │   (sviPredmetiItemId, all file IDs, initiatorEmail, orgUnitGroupId)
   │  │   → failure is non-fatal (warning logged, success still returned)
   │  │
   │  ├─ [IF varUsedReservedNumber = true]
   │  │   ├─ Delete RezervisaniBrojevi item
   │  │   └─ Log UseReservedNumber / Success
   │  │
   │  ├─ Log CreateDocument / Success
   │  └─ Respond success
   │
   ├─ [CATCH SCOPE — triggered if Terminate Failed was called]
   │   ├─ Log CreateDocument / Error / CREATE_DOCUMENT_FAILED
   │   └─ Respond failure
   │
   └─ Return to Canvas App
```

---

## Registry number acquisition paths

### Path A: Generate new number

Caller sets `useReservedNumber = false`. The flow calls CF_DocCentralV3_GenerateRegistryNumber.

- If the child flow succeeds: the counter in App Config has been atomically incremented.
  The number is now "owned" by this flow run. Filing date is set to today (UTC).
- If the child flow fails: no counter change was committed. Return failure to Canvas App. Clean state.

### Path B: Use reserved number

Caller sets `useReservedNumber = true` and provides `reservedNumberId` and `filingDate`.
The flow calls CF_DocCentralV3_UseReservedNumber.

- The child flow validates the reserved number but does NOT delete it.
- Deletion happens in Step 5l — after all creation steps succeed.
- If validation fails: return failure to Canvas App. Reserved number untouched.
- If creation fails after validation: reserved number is NOT deleted. It remains available.

---

## Filing date rules

| Scenario | Source | Set by |
|---|---|---|
| useReservedNumber = false | `utcNow('yyyy-MM-dd')` | This flow |
| useReservedNumber = true | `filingDate` input from caller | Canvas App |

When using a reserved number, the user explicitly selects the filing date in the Canvas App.
The flow does not override it with today's date.

---

## File system naming

All files are stored in the root of the Dokumenti library — no subfolders.

File name format:
- Main document: `{SafeDelovodniBroj}_{guid()}_{sanitizedOriginalFileName}`
- Attachment: `{SafeDelovodniBroj}_PRILOG_{guid()}_{sanitizedOriginalFileName}`

The `guid()` component guarantees uniqueness even if two files have the same original name
and the same delovodniBroj. No two files can overwrite each other.

The original file name is preserved in metadata (OriginalFileName column) for display
in the Canvas App and for audit purposes.

See `file-upload-pattern.md` for sanitization rules and file size limits.

---

## Metadata linking

All files uploaded for a document are linked to their Svi predmeti item via:
- `DelovodniBroj` column (same value as the Svi predmeti item)
- `SviPredmetiId` column (the integer item ID of the Svi predmeti item)

Attachments are additionally marked with:
- `IsPrilog = true`
- `ParentDelovodniBroj` = same delovodniBroj
- `ParentDocumentId` = same Svi predmeti item ID

This allows the Canvas App to query all files for a document using a filter on `SviPredmetiId`.

---

## Permission assignment

CF_DocCentralV3_AssignPermissions is called after all files are created.
All file IDs are accumulated in `varCreatedFileIds` during creation and passed in a single call.

The caller never has to call AssignPermissions separately — this flow handles it.

Permission failure is **non-fatal**: the document is created and the flow returns success
with a warning message. Administrators must repair permissions using the AuditLog entry.

See `PACode/flows/CF_DocCentralV3_AssignPermissions/consumption-pattern.md` — wait, that file
is in UseReservedNumber. See `PACode/flows/CF_DocCentralV3_AssignPermissions/permission-model.md`
for the full permission model.

---

## Non-fatal vs fatal failures

| Step | Fatal? | On failure |
|---|---|---|
| Input validation | Fatal — exit cleanly | Return VALIDATION_FAILED or similar. Nothing created. |
| App Config document type check | Fatal — exit cleanly | Return INVALID_DOCUMENT_TYPE. Nothing created. |
| Active year check | Fatal — exit cleanly | Return REGISTRY_YEAR_CLOSED. Nothing created. |
| GenerateRegistryNumber child flow | Fatal — exit cleanly | Return REGISTRY_NUMBER_GENERATION_FAILED. Nothing created. |
| UseReservedNumber child flow | Fatal — exit cleanly | Return RESERVED_NUMBER_INVALID. Nothing created. |
| Svi predmeti Create item | Fatal — Terminate Failed | Catch scope logs and returns CREATE_ITEM_FAILED. Number generated but not used. |
| Main file Create file | Fatal — Terminate Failed | Catch scope logs and returns CREATE_FILE_FAILED. Svi predmeti item exists orphaned. |
| Main file metadata update | Non-fatal — Warning | Document still valid. Metadata missing — admin fix required. |
| Attachment Create file | Non-fatal per attachment — Warning | Document still valid without that attachment. |
| Attachment metadata update | Non-fatal per attachment — Warning | File exists, metadata missing. |
| AssignPermissions child flow | Non-fatal — Warning | Document created, permissions incomplete. Success with warning returned. |
| Reserved number Delete item | Non-fatal — Warning | Document created, reserved number orphaned. Manual cleanup needed. |

---

## Audit log summary

| When | EventType | Status | Severity |
|---|---|---|---|
| Flow starts | CreateDocument | Started | Info |
| Input validation fails | ValidationError | Failed | Warning |
| Document type invalid | ValidationError | Failed | Warning |
| Registry year closed | CreateDocument | Failed | Error |
| Number generation fails | CreateDocument | Failed | Error |
| Reserved number invalid | CreateDocument | Failed | Error |
| Svi predmeti creation fails | CreateDocument | Failed | Error |
| File creation fails | CreateDocument | Failed | Error |
| Attachment fails | CreateDocument | — | Warning (non-fatal) |
| Permission failure | PermissionAssignment | Failed | Error (in AssignPermissions) |
| Reserved number deleted | UseReservedNumber | Success | Info |
| Full success | CreateDocument | Success | Info |
| Catch scope | CreateDocument | Failed | Error (CREATE_DOCUMENT_FAILED) |

---

## Callers and re-use

This flow is called exclusively by the Canvas App. It is not called by other flows.
No other flow should attempt to create a document item directly in Svi predmeti.

If a future requirement needs bulk document creation, a separate bulk flow should call
this flow in a loop — it must not bypass this flow and write directly to SharePoint.
