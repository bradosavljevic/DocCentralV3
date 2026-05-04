# Implementation steps: CF_DocCentralV3_CreateDocument

## Prerequisites

All foundation child flows must be built and published before this flow:
- CF_DocCentralV3_LogEvent
- CF_DocCentralV3_GenerateRegistryNumber
- CF_DocCentralV3_UseReservedNumber
- CF_DocCentralV3_AssignPermissions

SharePoint lists and library must exist:
- Svi predmeti (all columns confirmed — see `metadata-field-mapping.md`)
- Dokumenti document library (all columns confirmed — see `metadata-field-mapping.md`)
- RezervisaniBrojevi list
- App Config list with document type entries and active year entry

Connection references connected:
- CR_DocCentralV3_SharePoint
- CR_DocCentralV3_Office365Users (if used for user display name resolution)

Environment variables registered:
EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig,
EV_DocCentralV3_lstAuditLog, EV_DocCentralV3_docDokumenti, EV_DocCentralV3_lstRezervisaniBrojevi.

---

## Step 1 — Create the flow

1. Open the DocCentralV3 solution.
2. New → Cloud flow → Instant cloud flow.
3. Name: `CF_DocCentralV3_CreateDocument`.
4. Trigger: placeholder — replace immediately in Step 2.

---

## Step 2 — Replace trigger with Power Apps (V2)

This flow is called directly from the Canvas App (scrNoviPredmet screen).

1. Delete placeholder trigger.
2. Add trigger: **Power Apps (V2)**.
3. Add inputs:

| Input name | Type | Notes |
|---|---|---|
| documentType | Text | |
| useReservedNumber | Boolean | |
| reservedNumberId | Number | Pass 0 when not using reserved number |
| filingDate | Text | ISO 8601 date string; empty when not using reserved number |
| initiatorEmail | Text | |
| correlationId | Text | |
| metadataJson | Text | Full metadata object serialized as JSON string |
| mainFileJson | Text | `{fileName, fileContentBase64}` serialized as JSON string |
| attachmentsJson | Text | Array of `{fileName, fileContentBase64}` serialized as JSON string; pass `"[]"` if none |

**Why JSON strings for complex inputs:**
Power Apps V2 trigger does not support nested object or Array types. All structured inputs
(metadata, files, attachments) must be serialized as JSON strings in the Canvas App and parsed
inside the flow using `json()`.

Internal references (verify in expression editor after adding):
- `triggerBody()?['text_documentType']`
- `triggerBody()?['bool_useReservedNumber']`
- `triggerBody()?['decimal_reservedNumberId']`
- `triggerBody()?['text_filingDate']`
- `triggerBody()?['text_initiatorEmail']`
- `triggerBody()?['text_correlationId']`
- `triggerBody()?['text_metadataJson']`
- `triggerBody()?['text_mainFileJson']`
- `triggerBody()?['text_attachmentsJson']`

---

## Step 3 — Initialize variables

Add **Initialize variable** actions before any logic:

| Variable name | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varDelovodniBroj | String | `''` |
| varSafeDelovodniBroj | String | `''` |
| varSviPredmetiItemId | Integer | `0` |
| varMainFileSystemName | String | `''` |
| varMainFileId | Integer | `0` |
| varFilingDate | String | `''` |
| varUsedReservedNumber | Boolean | `false` |
| varReservedNumberItemId | Integer | `0` |
| varOrgUnitGroupId | String | `''` |
| varActiveYear | Integer | `0` |
| varMetadata | Object | `json(triggerBody()?['text_metadataJson'])` |
| varMainFile | Object | `json(triggerBody()?['text_mainFileJson'])` |
| varAttachments | Array | `json(if(empty(triggerBody()?['text_attachmentsJson']), '[]', triggerBody()?['text_attachmentsJson']))` |
| varCreatedFileIds | Array | `[]` (accumulates file item IDs for permission assignment) |

Initialize varCorrelationId first — all LogEvent calls depend on it.
Initialize varMetadata, varMainFile, varAttachments immediately after — needed in validation.

---

## Step 4 — Log Started

Add **Run a Child Flow** — CF_DocCentralV3_LogEvent:

| Parameter | Value |
|---|---|
| eventType | `CreateDocument` |
| eventCategory | `Document` |
| severity | `Info` |
| status | `Started` |
| correlationId | `variables('varCorrelationId')` |
| flowName | `CF_DocCentralV3_CreateDocument` |
| flowRunId | `workflow().run.name` |
| source | `CloudFlow` |
| userEmail | `triggerBody()?['text_initiatorEmail']` |
| message | `Pokretanje zavođenja dokumenta.` |

---

## Step 5 — Try scope

Add **Scope** action. Name it `Try`. All logic from Step 5a onward goes inside.

### 5a — Validate required inputs

Add a series of **Condition** actions. For each failed check: call LogEvent (ValidationError/Warning/Failed), Respond with failure, Terminate Succeeded.

Checks in order:
1. `empty(triggerBody()?['text_documentType'])` → VALIDATION_FAILED, message: `Tip dokumenta je obavezan.`
2. `empty(variables('varMetadata')?['subject'])` → VALIDATION_FAILED, message: `Predmet dokumenta je obavezan.`
3. `empty(variables('varMainFile')?['fileName'])` → VALIDATION_FAILED, message: `Naziv fajla je obavezan.`
4. `empty(variables('varMainFile')?['fileContentBase64'])` → VALIDATION_FAILED, message: `Sadržaj fajla je obavezan.`
5. `empty(triggerBody()?['text_initiatorEmail'])` → VALIDATION_FAILED, message: `Email inicijatora je obavezan.`
6. If `triggerBody()?['bool_useReservedNumber'] = true`:
   - `equals(int(triggerBody()?['decimal_reservedNumberId']), 0)` → VALIDATION_FAILED, message: `ID rezervisanog broja je obavezan pri korišćenju rezervisanog broja.`
   - `empty(triggerBody()?['text_filingDate'])` → VALIDATION_FAILED, message: `Datum zavođenja je obavezan pri korišćenju rezervisanog broja.`

### 5b — Read App Config: validate document type

Add **SharePoint — Get items**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter: UNKNOWN — filter by the document types configuration key
- Top: 50 (read all allowed types)

Rename: `Get_AppConfig_DocumentTypes`

Add Condition: check that `triggerBody()?['text_documentType']` appears in the returned values.
If not found: log ValidationError / INVALID_DOCUMENT_TYPE, Respond failure, Terminate Succeeded.

### 5c — Read App Config: active year and org unit group

Add **SharePoint — Get items** for the active year / registry book entry:
- Filter: UNKNOWN — active year filter key
- Top: 1

Rename: `Get_AppConfig_ActiveYear`

If empty: log Error / REGISTRY_YEAR_CLOSED, Respond failure, Terminate Succeeded.

Extract:
- Set varActiveYear = `int(first(body('Get_AppConfig_ActiveYear')?['value'])?['UNKNOWN_ActiveYearColumn'])`
- Set varOrgUnitGroupId from App Config (UNKNOWN key for org unit group mapped to documentType, or default group)

### 5d — Acquire registry number

Add **Condition**:
- Left: `triggerBody()?['bool_useReservedNumber']`
- Operator: is equal to
- Right: `true`

**Yes branch — use reserved number:**

1. Add **Run a Child Flow** — CF_DocCentralV3_UseReservedNumber:
   - reservedNumberId: `int(triggerBody()?['decimal_reservedNumberId'])`
   - requestedYear: `variables('varActiveYear')`
   - initiatorEmail: `triggerBody()?['text_initiatorEmail']`
   - correlationId: `variables('varCorrelationId')`

   Rename: `Call_UseReservedNumber`

2. Add Condition: `outputs('Call_UseReservedNumber')?['body']?['success']` equals `false`
   - Yes: Log RESERVED_NUMBER_INVALID / Error / Failed. Respond failure. Terminate Succeeded.

3. Set varDelovodniBroj = `outputs('Call_UseReservedNumber')?['body']?['delovodniBroj']`
4. Set varFilingDate = `triggerBody()?['text_filingDate']`
5. Set varUsedReservedNumber = `true`
6. Set varReservedNumberItemId = `int(triggerBody()?['decimal_reservedNumberId'])`

**No branch — generate new number:**

1. Add **Run a Child Flow** — CF_DocCentralV3_GenerateRegistryNumber:
   - initiatorEmail: `triggerBody()?['text_initiatorEmail']`
   - correlationId: `variables('varCorrelationId')`

   Rename: `Call_GenerateRegistryNumber`

2. Add Condition: `outputs('Call_GenerateRegistryNumber')?['body']?['success']` equals `false`
   - Yes: Log REGISTRY_NUMBER_GENERATION_FAILED / Error / Failed. Respond failure. Terminate Succeeded.

3. Set varDelovodniBroj = `outputs('Call_GenerateRegistryNumber')?['body']?['delovodniBroj']`
4. Set varFilingDate = `utcNow('yyyy-MM-dd')`

### 5e — Build SafeDelovodniBroj

Add **Compose** action. Name it `Build_SafeDelovodniBroj`.

Expression to sanitize varDelovodniBroj for use in file names:
```
replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(variables('varDelovodniBroj'), '/', '-'), '\\', '-'), ':', '-'), '*', ''), '?', ''), '"', ''), '<', ''), '>', ''), '|', ''), '  ', ' '), '.', ''), '#', ''), '%', '')
```

Then: Set varSafeDelovodniBroj = `trim(outputs('Build_SafeDelovodniBroj'))`

**Note:** SharePoint file name limits and forbidden characters are documented in `file-upload-pattern.md`. Review that document if sanitization produces empty strings.

### 5f — Create Svi predmeti item

Add **SharePoint — Create item**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`

Fields — see `metadata-field-mapping.md` for confirmed internal names. UNKNOWN names must be resolved before building:

| Display name | Value | Internal name |
|---|---|---|
| Title | `variables('varMetadata')?['subject']` | Title |
| DelovodniBroj | `variables('varDelovodniBroj')` | UNKNOWN |
| Stanje | `Zavedeno` | UNKNOWN |
| DocumentType | `triggerBody()?['text_documentType']` | UNKNOWN |
| FilingDate | `variables('varFilingDate')` | UNKNOWN |
| InicijatorEmail | `triggerBody()?['text_initiatorEmail']` | UNKNOWN |
| PartnerId | `variables('varMetadata')?['partnerId']` | UNKNOWN |
| PartnerNazivSnapshot | `variables('varMetadata')?['partnerNaziv']` | UNKNOWN |
| PartnerPIBSnapshot | `variables('varMetadata')?['partnerPIBSnapshot']` | UNKNOWN |
| PartnerMestoSnapshot | `variables('varMetadata')?['partnerMestoSnapshot']` | UNKNOWN |
| PartnerAdresaSnapshot | `variables('varMetadata')?['partnerAdresaSnapshot']` | UNKNOWN |
| CorrelationId | `variables('varCorrelationId')` | UNKNOWN |
| Description | `variables('varMetadata')?['description']` | UNKNOWN |

Rename action: `Create_SviPredmeti_Item`

Store: Set varSviPredmetiItemId = `int(outputs('Create_SviPredmeti_Item')?['body']?['ID'])`

Add Condition: if Create item status code is not 201 → Log CREATE_ITEM_FAILED / Error / Failed. See `rollback-and-compensation.md` for partial failure handling. Terminate Failed (triggers Catch).

### 5g — Generate main file system name

Add **Compose** action. Name it `Build_MainFileSystemName`.

Expression (sanitize original file name inline — or call a sub-expression):
```
concat(
  variables('varSafeDelovodniBroj'),
  '_',
  guid(),
  '_',
  <sanitized_mainFile_fileName>
)
```

Sanitizing the original file name uses the same character replacements as Step 5e.
Build a second Compose `Sanitize_MainFileName` first:
```
trim(replace(replace(replace(replace(replace(replace(replace(replace(variables('varMainFile')?['fileName'], '/', '-'), '\\', '-'), ':', '-'), '*', ''), '?', ''), '"', ''), '<', ''), '>', ''))
```

Then: Set varMainFileSystemName = output of `Build_MainFileSystemName`.

### 5h — Create main file in Dokumenti root

Add **SharePoint — Create file**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Folder Path: `/` + `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')`
  (root of the library — no subfolder)
- File Name: `variables('varMainFileSystemName')`
- File Content: `base64ToBinary(variables('varMainFile')?['fileContentBase64'])`

Rename: `Create_MainFile`

Store: Set varMainFileId = `int(outputs('Create_MainFile')?['body']?['ItemId'])`

Add Condition: if Create file fails → Log CREATE_FILE_FAILED / Error / Failed. See `rollback-and-compensation.md`. Terminate Failed.

Append varMainFileId to varCreatedFileIds:
```
union(variables('varCreatedFileIds'), array(variables('varMainFileId')))
```
Use Set variable with the union expression.

### 5i — Update main file metadata

Add **SharePoint — Update file properties**:
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Library Name: `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')`
- Id: `variables('varMainFileId')`

Fields — see `metadata-field-mapping.md` for confirmed internal names:

| Display name | Value | Internal name |
|---|---|---|
| DelovodniBroj | `variables('varDelovodniBroj')` | UNKNOWN |
| SviPredmetiId | `variables('varSviPredmetiItemId')` | UNKNOWN |
| IsPrilog | `false` | UNKNOWN |
| OriginalFileName | `variables('varMainFile')?['fileName']` | UNKNOWN |
| SystemFileName | `variables('varMainFileSystemName')` | UNKNOWN |
| DocumentType | `triggerBody()?['text_documentType']` | UNKNOWN |
| DocumentStatus | `Zavedeno` | UNKNOWN |
| CorrelationId | `variables('varCorrelationId')` | UNKNOWN |

Rename: `Update_MainFile_Metadata`

If update fails: log Warning (non-fatal — file exists, metadata is incomplete, admin can fix manually).

### 5j — Process attachments

Add **Condition**:
- Left: `length(variables('varAttachments'))`
- Operator: greater than
- Right: `0`

Yes branch: Add **Apply to each** over `variables('varAttachments')`.

Inside loop for each attachment:

1. **Sanitize attachment file name** — Compose:
   ```
   trim(replace(replace(replace(replace(variables('varMainFile')?['fileName'], '/', '-'), '\\', '-'), ':', '-'), '*', ''))
   ```
   (Apply same full sanitization as main file — reference `file-upload-pattern.md`.)
   Use `items('Apply_to_each_Attachments')?['fileName']` as source.

2. **Build attachment system name** — Compose:
   ```
   concat(variables('varSafeDelovodniBroj'), '_PRILOG_', guid(), '_', <sanitizedAttachmentFileName>)
   ```

3. **Create attachment file** — SharePoint Create file:
   - Folder path: root of Dokumenti library
   - File name: output of step 2
   - File content: `base64ToBinary(items('Apply_to_each_Attachments')?['fileContentBase64'])`

   Rename: `Create_AttachmentFile`

4. **Capture attachment file ID** — use a Compose or Set variable scoped to the loop:
   ```
   int(outputs('Create_AttachmentFile')?['body']?['ItemId'])
   ```
   Append to varCreatedFileIds using `union()`.

5. **Update attachment metadata** — SharePoint Update file properties:
   - Fields: DelovodniBroj, SviPredmetiId, IsPrilog=true, ParentDelovodniBroj, ParentDocumentId, OriginalFileName, SystemFileName, DocumentType, DocumentStatus, CorrelationId
   - All UNKNOWN internal names — see `metadata-field-mapping.md`

Attachment failures (file create or metadata update) are **non-fatal**. Log Warning per failed attachment. Continue loop. Document is still considered valid.

Rename loop: `Apply_to_each_Attachments`

### 5k — Assign permissions

Add **Run a Child Flow** — CF_DocCentralV3_AssignPermissions:
- sviPredmetiItemId: `variables('varSviPredmetiItemId')`
- documentLibraryFileIds: `string(variables('varCreatedFileIds'))` (serialized array)
- initiatorEmail: `triggerBody()?['text_initiatorEmail']`
- documentType: `triggerBody()?['text_documentType']`
- orgUnitGroupId: `variables('varOrgUnitGroupId')`
- additionalUserEmails: `'[]'`
- additionalGroupIds: `'[]'`
- correlationId: `variables('varCorrelationId')`

Rename: `Call_AssignPermissions`

Add Condition: if `outputs('Call_AssignPermissions')?['body']?['success']` = false:
- Log Warning: `Prava nisu dodeljena na dokument. Proverite audit log.` (non-fatal)
- Set a varPermissionsFailed = true flag for the response message

### 5l — Delete reserved number (if applicable)

Add **Condition**:
- Left: `variables('varUsedReservedNumber')`
- Operator: is equal to
- Right: `true`

Yes branch:
1. Add **SharePoint — Delete item**:
   - List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')`
   - ID: `variables('varReservedNumberItemId')`
   Rename: `Delete_ReservedNumber`

2. Add Condition: if Delete item fails → log Warning (non-fatal):
   - message: `Rezervisani broj nije obrisan automatski. Potrebno ručno brisanje.`
   - Continue flow.

3. If Delete succeeds: Log UseReservedNumber / Success / Info:
   - message: `concat('Rezervisani broj ', variables('varDelovodniBroj'), ' je iskorišćen i uklonjen.')`

### 5m — Log Success

Add **Run a Child Flow** — CF_DocCentralV3_LogEvent:
- eventType: `CreateDocument` / eventCategory: `Document`
- severity: `Info` / status: `Success`
- documentItemId: `variables('varSviPredmetiItemId')`
- delovodniBroj: `variables('varDelovodniBroj')`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Dokument ', variables('varDelovodniBroj'), ' je uspešno zaveden.')`

### 5n — Respond success

Add **Respond to a PowerApp or flow** — success outputs.
Include warning about permissions if varPermissionsFailed = true.

---

## Step 6 — Catch scope

Add **Scope** after Try. Name it `Catch`.
Configure Run after: uncheck `is successful`, check `has failed`, check `has timed out`.

Inside Catch:
1. Compose error details:
   ```
   result('Try')?[0]?['error']?['message']
   ```
2. Call CF_DocCentralV3_LogEvent:
   - eventType: `CreateDocument` / severity: `Error` / status: `Failed`
   - errorCode: `CREATE_DOCUMENT_FAILED`
   - errorMessage: composed error details
   - correlationId: `variables('varCorrelationId')`
   - message: `Greška pri zavođenju dokumenta. Pogledajte audit log za detalje.`
3. Respond failure.

---

## Step 7 — Post-scope routing

Add **Condition** after Try and Catch:
- Left: `result('Try')?[0]?['status']` equals `Succeeded`

Yes: Already responded in Step 5n — no further action needed.
No: Catch already responded — no further action needed.

---

## Step 8 — Respond output definitions

**Success:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `true` |
| message | Text | `Dokument je uspešno zaveden.` or `Dokument je zaveden ali dodela prava nije uspela.` |
| itemId | Integer | `variables('varSviPredmetiItemId')` |
| delovodniBroj | Text | `variables('varDelovodniBroj')` |
| correlationId | Text | `variables('varCorrelationId')` |
| errorCode | Text | `''` |

**Failure:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `false` |
| message | Text | Specific failure message |
| itemId | Integer | `0` |
| delovodniBroj | Text | `''` |
| correlationId | Text | `variables('varCorrelationId')` |
| errorCode | Text | Specific error code |

---

## Step 9 — Save and verify

1. Save frequently — this is the most complex flow in the solution.
2. Verify inside DocCentralV3 solution.
3. Run TC-001 through TC-005 before end-to-end Canvas App integration.
4. Description: `"v1.0 — initial implementation. CF_DocCentralV3_CreateDocument. DocCentralV3 solution."`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_CreateDocument`
- [ ] Trigger is Power Apps (V2) with 9 inputs (structured objects as JSON strings)
- [ ] All 15 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] JSON inputs parsed null-safely into variables
- [ ] Log Started called before Try scope
- [ ] All 7 validation checks performed; each failure exits with Terminate Succeeded
- [ ] App Config document type validation reads from list (not hardcoded list)
- [ ] App Config active year validation; REGISTRY_YEAR_CLOSED on failure
- [ ] Correct branch on useReservedNumber (true/false)
- [ ] UseReservedNumber failure exits cleanly
- [ ] GenerateRegistryNumber failure exits cleanly
- [ ] SafeDelovodniBroj built before file naming
- [ ] Svi predmeti item created; failure triggers Terminate Failed (Catch scope)
- [ ] Main file created in library root (no subfolder); failure triggers Terminate Failed
- [ ] varCreatedFileIds accumulates main file ID and all attachment file IDs
- [ ] Attachment failures are non-fatal (Warning log, loop continues)
- [ ] AssignPermissions called with serialized varCreatedFileIds array
- [ ] Permission failure is non-fatal (Warning log, success response with warning message)
- [ ] Reserved number deleted after all creation steps succeed (conditional)
- [ ] Reserved number deletion failure is non-fatal (Warning log)
- [ ] UseReservedNumber/Success log written after deletion
- [ ] Log Success called before Respond success
- [ ] Catch scope logs Error, returns structured failure response
- [ ] All 6 output parameters in both Respond actions
- [ ] All SharePoint actions use CR_DocCentralV3_SharePoint
- [ ] All EV references use @parameters() syntax
- [ ] All UNKNOWN column names resolved before building
- [ ] Flow is inside DocCentralV3 solution
