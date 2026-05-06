# 20 - Refactoring Backlog

Generated: 2026-05-06
Based on: docs/01–19, ai-instructions/*, flow-guides/*, sharepoint-guides/*

This backlog drives the DocCentral V3 refactor. Each item is scoped to a single, reviewable unit of work. No Canvas app formula changes may begin until the relevant phase 0 environment verification is complete and the item is explicitly approved.

---

## Legend

**Risk levels**
- `Critical` — data loss or number sequence corruption possible if done wrong
- `High` — incorrect implementation breaks a core business flow
- `Medium` — degraded UX or maintainability if skipped
- `Low` — quality improvement, no functional risk

**Manual guide required**
- `Yes` — Claude must produce a written implementation guide before human touches the component
- `No` — work is documentation-only or Canvas-only and does not touch backend

---

## Phase 1 — Documentation Alignment and Current-State Inventory

### BACK-001
**Title:** Confirm Phase 0 environment checklist and record results

**Affected screens:** None  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** All (verification only)

**Risk:** Critical  
**Manual guide required:** No

**Description:**  
Before any refactor work begins, the developer must verify and document the current state of every backend dependency. Unconfirmed components must not be assumed to exist.

**Acceptance criteria:**
- Checklist below is completed and all results recorded in a new `docs/21-environment-verification.md`
- Every confirmed component has its internal name, URL or ID recorded
- Every missing component is flagged with the guide that must be followed to create it

**Checklist:**
1. `NumberLocks` list exists with all columns from `sharepoint-guides/NumberLocks.md`; `LockKey` is unique + indexed
2. `AuditLog` list exists with all columns from `sharepoint-guides/AuditLog.md`
3. `TechnicalStatus` column exists in `Svi predmeti` with correct choice values
4. `DelovodniBroj` is indexed + unique in `Svi predmeti`
5. All `EV_DocCentralV3_*` environment variables created and point to correct SharePoint objects
6. All `CR_DocCentralV3_*` connection references exist and use service account
7. Internal column names returned for: `Svi predmeti`, `Dokumenta`, `NumberLocks`, `AuditLog`, `AppConfig`, `Rezervisani brojevi`, `Partneri`
8. Legacy flow inventory: which `CF_DocCentral21_*` flows are still active in production

**Test scenarios:**
- None — this is a verification and documentation task

---

### BACK-002
**Title:** Screen-by-screen audit of direct SharePoint writes in Canvas app

**Affected screens:** All  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** All

**Risk:** High  
**Manual guide required:** No

**Description:**  
Identify every direct `Patch`, `Remove`, `UpdateIf`, and `SubmitForm` against SharePoint data sources in the existing Canvas app. Without this inventory, Phase 2 refactor risks removing business logic silently.

**Acceptance criteria:**
- Audit document created as `docs/22-direct-sharepoint-write-audit.md`
- Every occurrence of `Patch`, `Remove`, `UpdateIf`, `SubmitForm` targeting SharePoint is listed with: screen name, control name, formula location (OnSelect / OnChange / etc.), target list/library, fields written, and business purpose
- Each entry is classified as: must migrate to flow / can remain as local collection patch / low risk
- Cross-screen control references are listed
- Non-delegable SharePoint queries are listed

**Test scenarios:**
- None — this is an analysis and documentation task

---

### BACK-003
**Title:** Confirm and document internal SharePoint column names

**Affected screens:** None  
**Affected flows:** All  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta, AppConfig, Rezervisani brojevi, NumberLocks, AuditLog, Partneri, EmailDocuments

**Risk:** High  
**Manual guide required:** No

**Description:**  
Power Automate OData filters and REST calls require internal column names, not display names. Claude must not guess these. Developer must return internal names for all critical columns in all affected lists. Results recorded in `sharepoint-guides/` files.

**Acceptance criteria:**
- Each `sharepoint-guides/` file updated with confirmed internal column names
- `TechnicalStatus` choice values confirmed
- `Stanje` choice values confirmed
- `NumberLocks.Status` choice values confirmed

**Test scenarios:**
- None — documentation task

---

### BACK-004
**Title:** Correct trigger type in scheduled flow guides

**Affected screens:** None  
**Affected flows:** CF_DocCentral_CleanupAuditLog, CF_DocCentral_EmailDocumentsCleanup  
**Affected SharePoint lists/libraries:** AuditLog, EmailDocuments

**Risk:** Medium  
**Manual guide required:** No

**Description:**  
`CF_DocCentral_CleanupAuditLog` and `CF_DocCentral_EmailDocumentsCleanup` are described as scheduled weekly flows, but both guides list trigger as "Power Apps V2." Scheduled flows require a Recurrence trigger. Guides must be corrected before implementation.

**Acceptance criteria:**
- Both flow guides updated with correct trigger type: Recurrence
- Schedule interval defined (weekly for AuditLog, configurable via AppConfig for EmailDocuments)
- Input section updated to remove Power Apps trigger inputs; add recurrence parameters

**Test scenarios:**
- None — documentation correction task

---

### BACK-005
**Title:** Define orchestration relationship between CreateDocumentTransaction and AssignRegistryNumber

**Affected screens:** scrAddDocument  
**Affected flows:** CF_DocCentral_CreateDocumentTransaction, CF_DocCentral_AssignRegistryNumber  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta, NumberLocks

**Risk:** Critical  
**Manual guide required:** No (design decision; guides updated after decision)

**Description:**  
The documentation does not specify whether `CF_DocCentral_CreateDocumentTransaction` calls `CF_DocCentral_AssignRegistryNumber` as a child flow, or whether they are independent flows called sequentially from Canvas. This must be decided before either guide is written to completion.

Decision options:
- A: Canvas calls `CreateDocumentTransaction` only; that flow internally calls/invokes number assignment logic
- B: Canvas calls `CreateDocumentTransaction` which creates the item, then separately calls `AssignRegistryNumber`
- C: Both flows are independent; Canvas orchestrates the sequence

**Recommended decision:** Option A — single Canvas call reduces partial-failure surface area. Canvas cannot safely sequence two critical flows without distributed transaction support.

**Acceptance criteria:**
- Decision recorded in `docs/05-appconfig-model.md` or a dedicated design note
- Flow guides for both flows updated to reflect the confirmed orchestration model
- Input/output schema for `CF_DocCentral_CreateDocumentTransaction` fully defined

**Test scenarios:**
- None — design decision task

---

### BACK-006
**Title:** Define approval mode configuration schema in AppConfig

**Affected screens:** scrProcess  
**Affected flows:** Approval-related flows  
**Affected SharePoint lists/libraries:** AppConfig

**Risk:** Medium  
**Manual guide required:** No

**Description:**  
`ProcesConfig` in AppConfig is referenced as the source of truth for approval, but the configurable approval mode (`App` vs `PowerAutomateApprovals`) has no defined schema. Before the approval process can be refactored, the schema must be documented.

**Acceptance criteria:**
- `docs/09-approval-process.md` updated with the `ProcesConfig` JSON schema including: `ApprovalMode` field with allowed values `App` / `PowerAutomateApprovals`, step definition structure, approver type (`User` / `Group`), active/inactive flag
- Example JSON included

**Test scenarios:**
- None — documentation task

---

### BACK-007
**Title:** Define archive book PDF generation mechanism

**Affected screens:** scrArchiveBook  
**Affected flows:** Archive book generation flow (not yet catalogued)  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta

**Risk:** High  
**Manual guide required:** Yes (after design decision)

**Description:**  
The archive book is described as a physical PDF stored in `Dokumenta` with its own `DelovodniBroj`. No flow, connector, or action for generating this PDF is described anywhere. The mechanism must be designed before the archive flow guide can be written.

Options to evaluate:
- Power Automate with HTML-to-PDF via OneDrive Convert action
- Power Automate with Word template and conversion
- Canvas export via PDF printer (not reliable at scale)

**Acceptance criteria:**
- PDF generation mechanism chosen and documented in `docs/10-archive-and-year-closing.md`
- Which Power Automate connector is used and whether it requires additional licensing confirmed
- Flow guide stub created for the archive book generation flow

**Test scenarios:**
- None — design decision task

---

### BACK-008
**Title:** Define reserved number escape hatch policy for year closing

**Affected screens:** scrLockYear  
**Affected flows:** Year closing flow  
**Affected SharePoint lists/libraries:** Rezervisani brojevi, AppConfig

**Risk:** High  
**Manual guide required:** No

**Description:**  
Current rule requires the reserved numbers list to be empty before year closing. If a reserved number is abandoned, no year can ever close. There is no documented admin override or exception policy.

**Acceptance criteria:**
- `docs/10-archive-and-year-closing.md` updated with: explicit policy on whether an admin can force-delete a reserved number before year close, whether an audit trail is required for such deletion, and whether the app must warn before allowing it

**Test scenarios:**
- None — policy documentation task

---

## Phase 2 — Remove or Isolate Direct SharePoint Writes from Canvas App

### BACK-009
**Title:** Migrate scrAddDocument direct SharePoint writes to CF_DocCentral_CreateDocumentTransaction

**Affected screens:** scrAddDocument  
**Affected flows:** CF_DocCentral_CreateDocumentTransaction  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta

**Risk:** Critical  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-005 must be complete. CF_DocCentral_CreateDocumentTransaction must be confirmed implemented and tested.

**Description:**  
`scrAddDocument` currently performs direct SharePoint writes for document creation and metadata. These must be replaced with a single call to `CF_DocCentral_CreateDocumentTransaction`. Canvas must not navigate away until a success response is received.

**Acceptance criteria:**
- No `Patch`, `Remove`, `UpdateIf`, or `SubmitForm` targeting SharePoint remains on `scrAddDocument`
- All write operations are performed via `CF_DocCentral_CreateDocumentTransaction`
- `Flow.Run` is wrapped in `IfError`
- Success response is checked before navigation
- User-friendly error message displayed on failure
- `TechnicalStatus` field reflected in UI for admin users on failure
- Main document file upload is part of the transaction (file reference passed to flow)
- Registration is blocked if no main document is selected

**Test scenarios:**
- REG-001: Register document with valid main file → item and file created, number assigned
- REG-002: Attempt registration without main file → blocked with user message
- REG-003: Flow returns error → error message displayed, no navigation, no partial item left visible
- REG-004: Register with optional attachment → attachment stored with `Attachment=true` and same `DelovodniBroj`
- REG-005: Simulate flow timeout → error caught, user informed, no number assigned silently

---

### BACK-010
**Title:** Migrate scrReserveNumber direct SharePoint writes to a reserve number flow

**Affected screens:** scrReserveNumber  
**Affected flows:** CF_DocCentral_ReserveNumber (new guide required)  
**Affected SharePoint lists/libraries:** Rezervisani brojevi

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Reserved number creation must go through Power Automate to enforce uniqueness validation and AuditLog. Direct write to `Rezervisani brojevi` from Canvas must be removed.

**Acceptance criteria:**
- No direct `Patch` to `Rezervisani brojevi` remains in `scrReserveNumber`
- Flow validates number is not already assigned in `Svi predmeti`
- Flow validates number is not already in `Rezervisani brojevi`
- Flow writes AuditLog entry
- Flow returns success/error response
- Canvas displays result and refreshes list

**Test scenarios:**
- RN-001: Reserve a valid number → number appears in list
- RN-002: Reserve a number already in `Svi predmeti` → blocked with message
- RN-003: Reserve a duplicate reserved number → blocked with message
- RN-004: Flow error → user informed, no partial write

---

### BACK-011
**Title:** Migrate scrPartneri direct SharePoint writes to CF_DocCentral partner flows

**Affected screens:** scrPartneri  
**Affected flows:** CF_DocCentral_AddPartner (new guide), CF_DocCentral_UpdatePartner (new guide), CF_DocCentral_DeletePartner (new guide)  
**Affected SharePoint lists/libraries:** Partneri

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Partner create, update, and delete operations must be moved to Power Automate. Delete must use hard delete (item to SharePoint recycle bin). Historical `ImePartnera` in `Svi predmeti` must not be affected.

**Acceptance criteria:**
- No direct `Patch`, `Remove` to `Partneri` remains in `scrPartneri`
- Delete flow performs hard delete only (no soft delete, no status column)
- Delete flow does not touch `Svi predmeti` records
- AuditLog entry written on delete
- Canvas refreshes partner collection via `CF_DocCentral_GetPartners_V2` after each write

**Test scenarios:**
- Add partner → appears in list
- Update partner name → `ImePartnera` in historical `Svi predmeti` items unchanged
- Delete partner → partner removed; existing `Svi predmeti` records retain `ImePartnera` value
- Delete partner → AuditLog entry created

---

### BACK-012
**Title:** Migrate scrProcess direct SharePoint writes to approval flows

**Affected screens:** scrProcess  
**Affected flows:** CF_DocCentral_ProcessApprovalResponse (new guide), CF_DocCentral_StartApproval (new guide)  
**Affected SharePoint lists/libraries:** Svi predmeti

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-006

**Description:**  
Any `Patch` updating `Stanje`, `TrenutniOdobravalacEmail`, `CurrentApprovalStep`, `ApprovalStepsJson`, or `ApprovalHistoryJson` directly from Canvas must be moved to Power Automate.

**Acceptance criteria:**
- No direct `Patch` to `Svi predmeti` from `scrProcess`
- Approval response sent via `CF_DocCentral_ProcessApprovalResponse`
- Flow updates all required fields atomically
- AuditLog entry written
- Initiator email notification sent on approval or rejection
- Canvas displays updated status after flow response

**Test scenarios:**
- APR-003: User approval → status updated, next step started or document becomes `Zavedeno`
- APR-005: Rejection → document becomes `Odbijeno`, initiator notified
- APR-006: Restart after rejection → initiator can edit metadata and resubmit

---

### BACK-013
**Title:** Migrate scrArchive direct SharePoint writes to archive flow

**Affected screens:** scrArchive  
**Affected flows:** CF_DocCentral_ArchiveDocument (new guide)  
**Affected SharePoint lists/libraries:** Svi predmeti

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Direct `Patch` setting `Stanje = Arhivirano` or archive metadata from Canvas must be removed. Archiving must go through Power Automate which validates the transition is allowed.

**Acceptance criteria:**
- No direct `Patch` to `Svi predmeti` from `scrArchive`
- Flow validates document is `Zavedeno` and not in process before archiving
- Flow writes AuditLog entry
- Canvas refreshes after flow response

**Test scenarios:**
- ARC-001: Archive `Zavedeno` document not in process → success
- ARC-002: Archive document currently in approval → blocked with message
- ARC-002b: Archive document with `Stanje != Zavedeno` → blocked

---

### BACK-014
**Title:** Isolate or remove remaining direct SharePoint writes across all other screens

**Affected screens:** scrHome, scrEmailDocs, scrDocSwap, scrAddReminders, scrSettings, scrCodes  
**Affected flows:** Various, to be identified in BACK-002  
**Affected SharePoint lists/libraries:** To be identified in BACK-002

**Risk:** High  
**Manual guide required:** Yes (per screen, based on BACK-002 findings)

**Dependencies:** BACK-002 must be complete

**Description:**  
After BACK-002 audit is complete, any remaining direct SharePoint writes on screens not covered by BACK-009 to BACK-013 must be assessed and migrated or removed.

**Acceptance criteria:**
- Zero `Patch`, `Remove`, `UpdateIf`, `SubmitForm` targeting SharePoint data sources anywhere in the Canvas app
- Each migrated write has a corresponding flow guide
- Each migration confirmed with the developer before formula changes

**Test scenarios:**
- Derived from BACK-002 audit findings

---

## Phase 3 — Registry Number / DelovodniBroj Safe Assignment Design

### BACK-015
**Title:** Write full implementation guide for CF_DocCentral_AssignRegistryNumber with NumberLocks

**Affected screens:** scrAddDocument  
**Affected flows:** CF_DocCentral_AssignRegistryNumber  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta, NumberLocks, Rezervisani brojevi, AppConfig, AuditLog

**Risk:** Critical  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-005

**Description:**  
Write the complete implementation guide for number assignment. Must specify the exact locking pattern. The safe pattern is: attempt to create a `NumberLocks` item with unique `LockKey` → if creation fails due to duplicate key, a lock exists → wait/fail. Do not use check-then-create (unsafe under concurrency).

Guide must include:
- Full input parameter list with types
- Full output JSON schema
- NumberLocks acquire/release/expire logic
- `FailedNeedsRecovery` detection and blocking logic
- No-skip validation (next number must equal max assigned + 1)
- Reserved number path (validate, use, delete)
- AuditLog entries for success, failure, recovery
- Recovery unblocking procedure

**Acceptance criteria:**
- Guide in `flow-guides/CF_DocCentral_AssignRegistryNumber.md` is complete with all sections from `docs/16-power-automate-manual-implementation-guide.md`
- Locking pattern uses unique-constraint-violation-catch, not check-then-create
- No-skip rule explicitly implemented in action sequence
- `FailedNeedsRecovery` blocks all new assignments until resolved

**Test scenarios:**
- REG-005: 10 concurrent registrations → 10 sequential unique numbers, no skips, no duplicates
- REG-006: Number assignment fails mid-transaction → `FailedNeedsRecovery` set, subsequent registrations blocked
- Recovery: admin clears `FailedNeedsRecovery` → registrations resume from correct next number
- Expired lock: lock older than 5 minutes → recovery validation required before continuation
- Reserved number path: reserved number used successfully → deleted from `Rezervisani brojevi` after both lists updated

---

### BACK-016
**Title:** Write full implementation guide for CF_DocCentral_CreateDocumentTransaction

**Affected screens:** scrAddDocument  
**Affected flows:** CF_DocCentral_CreateDocumentTransaction, CF_DocCentral_AssignRegistryNumber  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta, NumberLocks, AuditLog

**Risk:** Critical  
**Manual guide required:** Yes

**Dependencies:** BACK-005, BACK-015

**Description:**  
Write the complete implementation guide for the document registration transaction. This flow orchestrates: file upload to `Dokumenta`, metadata write to `Svi predmeti`, number assignment (calls or embeds `AssignRegistryNumber` logic), and AuditLog.

Guide must cover:
- File content input (base64 or file reference)
- All metadata inputs
- TechnicalStatus lifecycle: `DocumentCreationInProgress` → `NumberAssignmentInProgress` → `Completed` / `FailedNeedsRecovery`
- Attachment inputs (optional, one or more)
- Rollback / partial failure handling
- Recovery scenario for partial success states

**Acceptance criteria:**
- Guide complete with all sections per `docs/16-power-automate-manual-implementation-guide.md`
- TechnicalStatus updated at each transaction stage
- Partial failure leaves TechnicalStatus in a recoverable, detectable state
- AuditLog entry for success and each failure mode

**Test scenarios:**
- Successful registration with main file only
- Successful registration with main file + 1 attachment
- File upload succeeds but metadata write fails → `FailedNeedsRecovery`, no number assigned
- Metadata write succeeds but number assignment fails → `FailedNeedsRecovery`
- Both succeed but reserved number deletion fails → AuditLog warning, REC-003 recovery applies

---

### BACK-017
**Title:** Add FailedNeedsRecovery recovery UI for admin users

**Affected screens:** scrHome or scrSettings (to be decided)  
**Affected flows:** CF_DocCentral_RecoverTransaction (new guide required)  
**Affected SharePoint lists/libraries:** Svi predmeti, NumberLocks, AuditLog

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-016, BACK-001

**Description:**  
When `FailedNeedsRecovery` exists in `Svi predmeti`, the app must:
- Block registration for ordinary users with a clear message
- Show a recovery entry point for admin/consultant users
- Allow admin to trigger the recovery flow

**Acceptance criteria:**
- Ordinary users see a blocking message when `FailedNeedsRecovery` items exist
- Admin/consultant users see a list of affected items and a recovery action button
- Recovery flow guide written
- Recovery flow updates `TechnicalStatus` to `Completed` or `MetadataSyncFailed` depending on repair outcome
- AuditLog entry written for recovery action
- After successful recovery, registration is unblocked

**Test scenarios:**
- REC-001: Item exists with missing `DelovodniBroj` → admin sees it and triggers recovery → number assigned, status updated
- REC-002: `DelovodniBroj` in `Svi predmeti` but not in `Dokumenta` → recovery updates `Dokumenta` metadata
- Ordinary user during `FailedNeedsRecovery` → registration form blocked

---

### BACK-018
**Title:** Add unique index and enforcement for DelovodniBroj in Svi predmeti

**Affected screens:** None (SharePoint configuration)  
**Affected flows:** CF_DocCentral_AssignRegistryNumber  
**Affected SharePoint lists/libraries:** Svi predmeti

**Risk:** Critical  
**Manual guide required:** Yes (SharePoint guide)

**Dependencies:** BACK-003

**Description:**  
`DelovodniBroj` must be indexed and unique in `Svi predmeti`. If this is not already in place, a SharePoint guide must be written and the developer must configure it manually. This is the last line of defence against duplicate numbers if flow logic has a bug.

**Acceptance criteria:**
- `sharepoint-guides/SharePointIndexesAndUniqueConstraints.md` updated to confirm the index exists
- Unique constraint configured in SharePoint list settings
- Test confirming that attempting to insert a duplicate `DelovodniBroj` fails at the SharePoint level

**Test scenarios:**
- Attempt to create two `Svi predmeti` items with the same `DelovodniBroj` via flow test → second insert fails at SharePoint
- Normal registration → succeeds and `DelovodniBroj` is assigned correctly

---

## Phase 4 — Large List Loading Flows and Canvas Collection Loading

### BACK-019
**Title:** Write and implement CF_DocCentral_GetPartners_V2

**Affected screens:** scrAddDocument, scrPartneri, scrHome  
**Affected flows:** CF_DocCentral_GetPartners_V2  
**Affected SharePoint lists/libraries:** Partneri

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Replace direct Canvas gallery binding to `Partneri` with a flow-based JSON collection. Flow uses SharePoint REST API with `$select` for required columns, `$top=5000`, and skiptoken pagination if list exceeds 5000 items. Canvas loads collection on screen load and caches in `colSp_Partneri`.

**Acceptance criteria:**
- Flow guide complete per `docs/16-power-automate-manual-implementation-guide.md`
- REST query selects only required columns: ID, Title, Adresa, Grad, TipKontakta, PIB, JMBG, VrstaKontakta, MaticniBroj, BCID, Klasifikacija, Modified
- Response envelope: `{ success, code, message, dataJson, count }`
- Canvas parses JSON into `colSp_Partneri` on flow success
- Error displayed if flow returns `success: false`
- No direct SharePoint `Partneri` datasource binding remains on any screen

**Test scenarios:**
- Partneri list with 100 items → loaded correctly
- Partneri list with 5100 items → all items loaded via pagination
- Flow error → empty collection, error message displayed, no crash

---

### BACK-020
**Title:** Write and implement CF_DocCentral_GetDocumentsForArchiving_V2

**Affected screens:** scrArchive  
**Affected flows:** CF_DocCentral_GetDocumentsForArchiving_V2  
**Affected SharePoint lists/libraries:** Svi predmeti

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Replace direct Canvas query against `Svi predmeti` for archiving candidates with a flow returning a filtered JSON collection. Filter: `Stanje eq 'Zavedeno'` AND not in process AND active year AND not already archived.

**Acceptance criteria:**
- Flow guide complete
- OData filter defined and tested
- Only required columns selected
- Pagination supported for large datasets
- Response envelope standard
- Canvas loads into `colSp_DocsForArchiving` and handles empty result and error cases

**Test scenarios:**
- Active year has 200 eligible documents → all returned
- Active year has 2500 eligible documents → all returned via pagination
- No eligible documents → empty collection, no error
- Flow error → error message displayed

---

### BACK-021
**Title:** Write and implement CF_DocCentral_GetSviPredmeti (filtered read for scrHome)

**Affected screens:** scrHome  
**Affected flows:** CF_DocCentral_GetSviPredmeti (new guide)  
**Affected SharePoint lists/libraries:** Svi predmeti

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
`scrHome` likely loads a view of recent or filtered documents from `Svi predmeti`. If this list exceeds 2000 items the delegation limit will be hit. Replace with a read flow that accepts filter parameters (date range, status, organizational unit) and returns a paged JSON response.

**Acceptance criteria:**
- Flow accepts filter inputs: `dateFrom`, `dateTo`, `stanje`, `organizacionaJedinica` (all optional)
- Returns paged JSON with standard response envelope
- Canvas uses `colSp_SviPredmeti` and shows loading indicator during flow call
- No direct `Svi predmeti` gallery data source used for the main document list on scrHome

**Test scenarios:**
- Load last 30 days, status = Zavedeno → filtered results
- Load with no filters → returns paged first 500 items (or configurable page size)
- Over 2000 matching items → pagination works

---

### BACK-022
**Title:** Write and implement CF_DocCentral_GetEmailDocuments

**Affected screens:** scrEmailDocs  
**Affected flows:** CF_DocCentral_GetEmailDocuments (new guide)  
**Affected SharePoint lists/libraries:** EmailDocuments

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-004 (EmailDocuments status column must be defined)

**Description:**  
`scrEmailDocs` loads unprocessed email documents. `EmailDocuments` is a library that may grow. Replace direct Canvas binding with a flow-based filtered read returning only unprocessed items.

**Acceptance criteria:**
- Flow filters on processed/status column (after BACK-004 defines it)
- Returns only unprocessed items
- Canvas loads into `colSp_EmailDocuments`
- Error and empty state handled

**Test scenarios:**
- 50 unprocessed items → all returned
- 0 unprocessed items → empty state, no error
- Flow error → error message

---

## Phase 5 — AppConfig Administration Refactor

### BACK-023
**Title:** Write full implementation guide for CF_DocCentral_UpdateAppConfig

**Affected screens:** scrCodes, scrSettings  
**Affected flows:** CF_DocCentral_UpdateAppConfig  
**Affected SharePoint lists/libraries:** AppConfig, AuditLog

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003

**Description:**  
Write the complete guide for the AppConfig update flow. Flow must: receive config key and new JSON payload, validate JSON schema against the expected structure for that key, compare old and new values, write to AppConfig item via service account, and log old JSON, new JSON, hash, and changed-by to AuditLog.

Guide must include schema validation expressions per key (at minimum structural check — required fields present, correct types).

**Acceptance criteria:**
- Guide complete per `docs/16-power-automate-manual-implementation-guide.md`
- Schema validation rejects missing required fields
- Schema validation rejects wrong types on critical fields
- AuditLog includes ConfigKey, OldJson, NewJson, ChangedBy, ChangedAt, ValidationResult
- Flow returns clear rejection message with validation error detail to Canvas

**Test scenarios:**
- Valid JSON update → AppConfig item updated, AuditLog written
- Invalid JSON (missing required field) → rejected, error returned, no write
- Valid update to `EfaktureParametri` → succeeds
- Attempt to update legacy `EFaktureParametri` → flow should warn but not block (migration path documented)

---

### BACK-024
**Title:** Refactor scrCodes into per-dictionary sections

**Affected screens:** scrCodes  
**Affected flows:** CF_DocCentral_UpdateAppConfig  
**Affected SharePoint lists/libraries:** AppConfig

**Risk:** Medium  
**Manual guide required:** No (Canvas refactor only, after BACK-023 flow confirmed)

**Dependencies:** BACK-023 flow confirmed

**Description:**  
`scrCodes` is too large and hard to maintain. Refactor into a tabbed or navigable layout where each AppConfig dictionary is managed independently. Each section shows the current JSON records, allows adding/editing rows, validates schema before submitting, and calls `CF_DocCentral_UpdateAppConfig`.

**Acceptance criteria:**
- Each AppConfig key has its own management section
- Adding/editing a row validates at Canvas level before calling flow
- Flow is called with full updated JSON for the affected key
- Import from Excel supported for each dictionary independently
- Export to Excel supported
- No direct `Patch` to AppConfig from Canvas

**Test scenarios:**
- Edit a `TipoviDokumenta` entry → change saved via flow, list refreshed
- Add new `OrganizacionaJedinice` entry with duplicate ID → validation blocks at Canvas level
- Excel import of `VrsteDokumenta` with 50 rows → new rows inserted, changed rows updated, unchanged rows untouched
- Invalid Excel (missing required column) → import blocked with message

---

### BACK-025
**Title:** Implement Excel import/export for AppConfig dictionaries

**Affected screens:** scrCodes  
**Affected flows:** CF_DocCentral_ImportAppConfigFromExcel (new guide), CF_DocCentral_ExportAppConfigToExcel (new guide)  
**Affected SharePoint lists/libraries:** AppConfig, OneDrive/Exports

**Risk:** Medium  
**Manual guide required:** Yes

**Dependencies:** BACK-023, BACK-024

**Description:**  
Each AppConfig dictionary should support import from Excel and export to Excel. Import logic: insert missing rows (by ID), update changed rows, leave unchanged rows, reject rows with invalid schema, log all changes.

**Acceptance criteria:**
- Import flow validates each row before writing
- Import does not do a blind replace of the entire JSON
- Export produces a clean Excel with one row per JSON record
- Import/export available per dictionary key from scrCodes
- AuditLog entry for each import operation with count of inserted, updated, unchanged, rejected rows

**Test scenarios:**
- Import 100-row Excel → correct delta applied
- Import Excel with 3 invalid rows → invalid rows rejected, valid rows applied, rejection detail returned
- Export → Excel downloaded with current JSON records

---

## Phase 6 — Approval Process Cleanup

### BACK-026
**Title:** Write implementation guide for CF_DocCentral_StartApproval

**Affected screens:** scrAddDocument, scrProcess  
**Affected flows:** CF_DocCentral_StartApproval (new guide)  
**Affected SharePoint lists/libraries:** Svi predmeti, AuditLog

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-006

**Description:**  
When a document is registered and `ProcesConfig` has an active process for its `TipDokumenta`, the approval process must start automatically. Write the implementation guide for the flow that reads `ProcesConfig`, resolves the first step, updates `Svi predmeti` fields (`Stanje`, `TrenutniOdobravalacEmail`, `CurrentApprovalStep`, `TotalApprovalSteps`, `ApprovalStepsJson`), and sends notifications.

**Acceptance criteria:**
- Flow reads `ProcesConfig` for the given `TipDokumenta`
- If no active process, document status set to `Zavedeno` — no approval started
- If active process, first approver resolved (user or group)
- `Svi predmeti` updated with approval step metadata
- Email notification sent to approver(s)
- AuditLog entry written
- Guide supports both `App` and `PowerAutomateApprovals` modes

**Test scenarios:**
- APR-001: `TipDokumenta` has active process → document enters process, approver notified
- APR-002: `TipDokumenta` has no active process → document set to `Zavedeno`, no notification
- Group approval → all group members notified

---

### BACK-027
**Title:** Write implementation guide for CF_DocCentral_ProcessApprovalResponse

**Affected screens:** scrProcess  
**Affected flows:** CF_DocCentral_ProcessApprovalResponse  
**Affected SharePoint lists/libraries:** Svi predmeti, AuditLog

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-026

**Description:**  
Write the guide for processing an approval or rejection response. Flow must handle: advance to next step or complete to `Zavedeno` on approval; set `Odbijeno` and notify initiator on rejection; first-valid-response-wins for group approvals; update `ApprovalHistoryJson`; write AuditLog.

**Acceptance criteria:**
- Approval advances step counter; on final step, sets `Stanje = Zavedeno`
- Rejection sets `Stanje = Odbijeno`, sends email to `InitiatorEmail`
- Group approval: second response after first is already processed → ignored gracefully
- `ApprovalHistoryJson` updated with response detail and timestamp
- AuditLog entry written

**Test scenarios:**
- APR-003: Single user approves → next step or `Zavedeno`
- APR-004: Group — first member approves → step completed; second member approves → ignored
- APR-005: Rejection → `Odbijeno`, initiator notified
- APR-006: Initiator edits and restarts → process restarts from step 1

---

### BACK-028
**Title:** Approval history retention policy implementation

**Affected screens:** scrProcess  
**Affected flows:** None additional  
**Affected SharePoint lists/libraries:** Svi predmeti, AuditLog

**Risk:** Low  
**Manual guide required:** No

**Description:**  
Resolve the contradiction identified in BACK-006 / C-3: `ApprovalHistoryJson` in `Svi predmeti` has no cleanup mechanism. Decide whether history lives in the column permanently (acceptable if archived items are moved out of active view) or whether it should be moved to AuditLog only and the column cleared after process completion.

**Acceptance criteria:**
- Policy documented in `docs/09-approval-process.md`
- If `ApprovalHistoryJson` is kept in `Svi predmeti`, document that it is permanent and intentional
- If moved to AuditLog, document the migration point (when column is cleared)

**Test scenarios:**
- None — documentation/policy task

---

## Phase 7 — Archiving and Year Closing Cleanup

### BACK-029
**Title:** Write implementation guide for CF_DocCentral_GenerateArchiveBook

**Affected screens:** scrArchiveBook  
**Affected flows:** CF_DocCentral_GenerateArchiveBook (new guide)  
**Affected SharePoint lists/libraries:** Svi predmeti, Dokumenta, AuditLog

**Risk:** High  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-007 (PDF mechanism decided)

**Description:**  
Write the implementation guide for archive book generation. Flow must: verify no archive book exists for the year; retrieve all archived documents for the year; generate a PDF; upload to `Dokumenta`; create `Svi predmeti` item with own `DelovodniBroj`; write AuditLog. Duplicate generation must be blocked.

**Acceptance criteria:**
- Flow checks for existing archive book for the year before starting
- Duplicate generation returns clear error, no partial write
- Archive book registered in `Svi predmeti` with its own `DelovodniBroj`
- `DelovodniBroj` assigned through normal number assignment path (same counter as other documents)
- AuditLog entry written

**Test scenarios:**
- ARC-003: Generate for valid year → PDF created, registered correctly
- ARC-004: Generate again for same year → blocked with error
- REC-004: PDF created but registration fails → AuditLog, recovery flow completes registration

---

### BACK-030
**Title:** Write implementation guide for CF_DocCentral_CloseYear

**Affected screens:** scrLockYear  
**Affected flows:** CF_DocCentral_CloseYear (new guide)  
**Affected SharePoint lists/libraries:** AppConfig, Svi predmeti, Rezervisani brojevi, AuditLog

**Risk:** Critical  
**Manual guide required:** Yes

**Dependencies:** BACK-001, BACK-003, BACK-008, BACK-029

**Description:**  
Write the implementation guide for year closing. Flow must validate all prerequisites before making any change: all documents archived, no documents in process or rejected, reserved numbers list empty, archive book exists. If any prerequisite fails, return specific blocking reason. If all pass: update existing `DelovodneKnjige` JSON in AppConfig (do not create new item), set new active year, reset counter to 1.

**Acceptance criteria:**
- Each prerequisite checked independently with a distinct error code returned on failure
- No changes made if any prerequisite fails
- Year closing updates AppConfig JSON in place — no new `DelovodneKnjige` item
- Counter reset to 1 for new year
- Closed year flag set and irreversible
- AuditLog entry written

**Test scenarios:**
- YEAR-001: Non-archived documents exist → blocked, specific message
- YEAR-002: Reserved numbers exist → blocked, specific message
- YEAR-003: All prerequisites met → year closed, counter reset
- YEAR-004: Attempt to reopen closed year → not possible (no flow for this; UI must not offer the action)
- YEAR-003b: Attempt to close already-closed year → blocked

---

### BACK-031
**Title:** Write implementation guide for CF_DocCentral_CleanupAuditLog

**Affected screens:** None (scheduled)  
**Affected flows:** CF_DocCentral_CleanupAuditLog  
**Affected SharePoint lists/libraries:** AuditLog

**Risk:** Low  
**Manual guide required:** Yes

**Dependencies:** BACK-004 (trigger corrected to Recurrence)

**Description:**  
Scheduled weekly flow that deletes AuditLog entries older than 6 months. Must use batched delete to avoid SharePoint throttling. Must write a summary AuditLog entry after cleanup (entries deleted count).

**Acceptance criteria:**
- Recurrence trigger: weekly
- OData filter: `Created lt [today minus 6 months]`
- Batched delete (Apply to each with throttle management)
- Summary AuditLog entry written after completion
- Flow does not fail silently if batch delete fails

**Test scenarios:**
- AuditLog with entries older than 6 months → old entries deleted, recent entries kept
- AuditLog with no old entries → flow completes with 0 deletions, no error
- Flow error during batch delete → error logged, partial deletion is acceptable

---

### BACK-032
**Title:** Write implementation guide for CF_DocCentral_EmailDocumentsCleanup

**Affected screens:** None (scheduled)  
**Affected flows:** CF_DocCentral_EmailDocumentsCleanup  
**Affected SharePoint lists/libraries:** EmailDocuments, AppConfig

**Risk:** Low  
**Manual guide required:** Yes

**Dependencies:** BACK-004 (trigger corrected), BACK-022 (EmailDocuments status column defined)

**Description:**  
Scheduled flow that deletes processed `EmailDocuments` files older than the configured retention period. Retention period read from AppConfig (recommended default: 30 days). Only delete files marked as processed.

**Acceptance criteria:**
- Recurrence trigger: configurable (default daily or weekly)
- Retention period read from AppConfig `Settings` key
- Only processed files deleted
- Unprocessed files never touched
- AuditLog summary entry written

**Test scenarios:**
- EML-003: Processed files older than retention → deleted
- Unprocessed files regardless of age → not deleted
- Flow error → AuditLog warning, no data loss (REC-006 applies)

---

## Phase 8 — Responsive UI and AppChecker Cleanup

### BACK-033
**Title:** Apply responsive layout to all screens

**Affected screens:** All  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** None

**Risk:** Low  
**Manual guide required:** No

**Dependencies:** All Phase 2–7 screen refactors complete (do not mix layout and logic changes)

**Description:**  
Apply responsive layout formulas to all screens so the app works correctly on desktop, tablet, and mobile. Use container-based layout. App must also work embedded in SharePoint (fixed header height, responsive width).

**Acceptance criteria:**
- All screens render without horizontal scroll on 375px width (mobile)
- All screens render correctly at 768px (tablet) and 1280px (desktop)
- App embedded in SharePoint iframe at standard SharePoint web part width renders correctly
- No controls have hard-coded pixel positions that break at other form factors
- Existing users can recognise the layout (minimal redesign)

**Test scenarios:**
- Open app on mobile device → all screens usable
- Open app in SharePoint web part → no overflow, no scroll
- Open app on desktop → full-width layout used

---

### BACK-034
**Title:** Implement Serbian and English localization via Translations AppConfig key

**Affected screens:** All  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** AppConfig

**Risk:** Medium  
**Manual guide required:** No

**Dependencies:** AppConfig `Translations` key must be populated with all UI strings; BACK-023 flow must be available for updates

**Description:**  
All user-visible strings must be served from the `Translations` AppConfig key. Canvas loads translations on start into `colCfg_Translations`. All labels, button text, messages, and error strings use a `LookUp(colCfg_Translations, Key = "...", Value)` pattern. Language selection stored in `gbl_Language`.

**Acceptance criteria:**
- No hardcoded Serbian or English string remains in any formula (except fallback/debug strings)
- Language toggle available (user preference)
- All error messages from flows translated
- All screen titles, labels, and button text translated
- Missing translation key shows key name as fallback (not blank, not crash)

**Test scenarios:**
- Switch language to English → all UI strings change
- Switch language to Serbian → all UI strings change
- Missing translation key → fallback key name shown, no error

---

### BACK-035
**Title:** Resolve AppChecker findings — accessibility labels

**Affected screens:** All  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** None

**Risk:** Low  
**Manual guide required:** No

**Dependencies:** BACK-033 (layout), BACK-034 (localization)

**Description:**  
AppChecker flags controls missing accessibility labels (`AccessibleLabel`). All interactive controls (buttons, inputs, galleries, dropdowns, toggles, date pickers) must have `AccessibleLabel` set to a translated string from `colCfg_Translations`.

**Acceptance criteria:**
- Zero AppChecker errors for missing `AccessibleLabel`
- All `AccessibleLabel` values use localization pattern, not hardcoded strings

**Test scenarios:**
- Run AppChecker → zero accessibility label errors

---

### BACK-036
**Title:** Resolve AppChecker findings — TabIndex and focus management

**Affected screens:** All  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** None

**Risk:** Low  
**Manual guide required:** No

**Dependencies:** BACK-033

**Description:**  
AppChecker flags non-standard `TabIndex` values. All interactive controls must have logical tab order within each screen. Non-interactive labels and decorative controls should have `TabIndex = -1`.

**Acceptance criteria:**
- Zero AppChecker errors for TabIndex
- Keyboard navigation follows logical visual order on each screen

**Test scenarios:**
- Tab through scrAddDocument using keyboard only → logical navigation order

---

### BACK-037
**Title:** Resolve AppChecker findings — formula complexity and screen complexity

**Affected screens:** scrCodes, scrAddDocument, scrProcess (highest complexity expected)  
**Affected flows:** None  
**Affected SharePoint lists/libraries:** None

**Risk:** Low  
**Manual guide required:** No

**Dependencies:** BACK-024 (scrCodes refactor), Phase 2 screen refactors

**Description:**  
AppChecker flags screens and formulas that are too complex. After Phase 2 and Phase 5 refactors, residual complexity issues should be addressed. Use `With()` to flatten complex formulas. Split large screens into components if needed.

**Acceptance criteria:**
- Zero AppChecker errors for formula or screen complexity
- No formula exceeds readable complexity without a documented reason

**Test scenarios:**
- Run AppChecker after each screen refactor → complexity score improves

---

## Backlog Summary Table

| ID | Phase | Title | Risk | Guide required |
|---|---|---|---|---|
| BACK-001 | 1 | Confirm Phase 0 environment checklist | Critical | No |
| BACK-002 | 1 | Screen-by-screen direct write audit | High | No |
| BACK-003 | 1 | Confirm internal SharePoint column names | High | No |
| BACK-004 | 1 | Correct scheduled flow trigger types | Medium | No |
| BACK-005 | 1 | Define orchestration: CreateDocumentTransaction vs AssignRegistryNumber | Critical | No |
| BACK-006 | 1 | Define approval mode configuration schema | Medium | No |
| BACK-007 | 1 | Define archive book PDF generation mechanism | High | Yes |
| BACK-008 | 1 | Define reserved number escape hatch policy | High | No |
| BACK-009 | 2 | Migrate scrAddDocument writes to CreateDocumentTransaction | Critical | Yes |
| BACK-010 | 2 | Migrate scrReserveNumber writes to reserve number flow | High | Yes |
| BACK-011 | 2 | Migrate scrPartneri writes to partner flows | Medium | Yes |
| BACK-012 | 2 | Migrate scrProcess writes to approval flows | High | Yes |
| BACK-013 | 2 | Migrate scrArchive writes to archive flow | High | Yes |
| BACK-014 | 2 | Isolate remaining direct writes on other screens | High | Yes |
| BACK-015 | 3 | Full guide: CF_DocCentral_AssignRegistryNumber | Critical | Yes |
| BACK-016 | 3 | Full guide: CF_DocCentral_CreateDocumentTransaction | Critical | Yes |
| BACK-017 | 3 | FailedNeedsRecovery recovery UI for admin | High | Yes |
| BACK-018 | 3 | Unique index on DelovodniBroj in Svi predmeti | Critical | Yes |
| BACK-019 | 4 | CF_DocCentral_GetPartners_V2 | Medium | Yes |
| BACK-020 | 4 | CF_DocCentral_GetDocumentsForArchiving_V2 | Medium | Yes |
| BACK-021 | 4 | CF_DocCentral_GetSviPredmeti | Medium | Yes |
| BACK-022 | 4 | CF_DocCentral_GetEmailDocuments | Medium | Yes |
| BACK-023 | 5 | Full guide: CF_DocCentral_UpdateAppConfig | High | Yes |
| BACK-024 | 5 | Refactor scrCodes into per-dictionary sections | Medium | No |
| BACK-025 | 5 | Excel import/export for AppConfig dictionaries | Medium | Yes |
| BACK-026 | 6 | Guide: CF_DocCentral_StartApproval | High | Yes |
| BACK-027 | 6 | Guide: CF_DocCentral_ProcessApprovalResponse | High | Yes |
| BACK-028 | 6 | Approval history retention policy | Low | No |
| BACK-029 | 7 | Guide: CF_DocCentral_GenerateArchiveBook | High | Yes |
| BACK-030 | 7 | Guide: CF_DocCentral_CloseYear | Critical | Yes |
| BACK-031 | 7 | Guide: CF_DocCentral_CleanupAuditLog | Low | Yes |
| BACK-032 | 7 | Guide: CF_DocCentral_EmailDocumentsCleanup | Low | Yes |
| BACK-033 | 8 | Responsive layout all screens | Low | No |
| BACK-034 | 8 | Serbian/English localization via Translations | Medium | No |
| BACK-035 | 8 | AppChecker: accessibility labels | Low | No |
| BACK-036 | 8 | AppChecker: TabIndex and focus | Low | No |
| BACK-037 | 8 | AppChecker: formula and screen complexity | Low | No |
