# Flow design: CF_DocCentralV3_ArchiveDocument

## Purpose

Archives one or more documents by transitioning their `Stanje` from `Zavedeno` to `Arhivirano`.
Applies the selected archive sign (`ArhivskiZnak`) to each document.
Updates the document item in `Svi predmeti`.
Logs each archiving event to `AuditLog`.

Direct transition: `Zavedeno` → `Arhivirano` is the only permitted direct archiving path.

## Trigger type

Power Apps V2. Called directly from Canvas App (archiving screen).

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

Supports archiving multiple documents in a single call to reduce flow invocations.

```json
{
  "documents": [
    {
      "documentItemId": 0,
      "delovodniBroj": "",
      "arhivskiZnak": ""
    }
  ],
  "initiatorEmail": "",
  "correlationId": ""
}
```

- `documents`: array of documents to archive. Must contain at least one item.
- `arhivskiZnak`: archive classification code — must be a valid value from App Config (UNKNOWN key for archive signs list).
- `initiatorEmail`: user performing the archiving action.

## Output schema

Success (all archived):
```json
{
  "success": true,
  "message": "Dokumenti su uspešno arhivirani.",
  "archivedCount": 0,
  "failedCount": 0,
  "failures": [],
  "correlationId": ""
}
```

Partial success:
```json
{
  "success": false,
  "message": "Deo dokumenata nije arhiviran.",
  "archivedCount": 0,
  "failedCount": 0,
  "failures": [
    {
      "documentItemId": 0,
      "delovodniBroj": "",
      "errorCode": "",
      "message": ""
    }
  ],
  "correlationId": ""
}
```

The flow returns `success: false` if any document fails to archive, even if others succeed.
This allows the Canvas App to display which documents failed.

## Pre-conditions per document

Each document in the array is validated individually before any update is made.

| Check | Failure code |
|---|---|
| Document exists in Svi predmeti | DOC_NOT_FOUND |
| Document Stanje is `Zavedeno` | INVALID_STATUS_FOR_ARCHIVE |
| `arhivskiZnak` is valid per App Config | INVALID_ARCHIVE_SIGN |

## Flow steps

### Initialization

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varArchivedCount` (Integer) = 0
- `varFailedCount` (Integer) = 0
- `varFailures` (Array) = []

**Step 2 — Validate input**
If `documents` array is empty: return failure immediately with code `NO_DOCUMENTS_PROVIDED`.

**Step 3 — Read valid archive signs from App Config**
Action: SharePoint — Get items from App Config filtered by archive sign category (UNKNOWN key).
Store in `varValidArchiveSigns` array of allowed sign values.

**Step 4 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ArchiveDocument`
- EventCategory: `Archive`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Pokretanje arhiviranja ", string(length(documents)), " dokumenata.")`

### Per-document processing loop

**Step 5 — Apply to each document in `documents` array**

For each document item, within a scoped error handler (not the global Catch):

**Step 5a — Validate archive sign**
Check that document `arhivskiZnak` is in `varValidArchiveSigns`.
If not: add to `varFailures`. Increment `varFailedCount`. Continue to next document.

**Step 5b — Read document item from Svi predmeti**
Action: SharePoint — Get item
ID: `documentItemId`

If not found: add to `varFailures` (DOC_NOT_FOUND). Increment `varFailedCount`. Continue.

Store `varCurrentStanje` = item `Stanje` value.
Store `varItemETag` = item `@odata.etag`.

**Step 5c — Validate current Stanje**
If `varCurrentStanje` is not `Zavedeno`:
Add to `varFailures` (INVALID_STATUS_FOR_ARCHIVE, message including actual status).
Increment `varFailedCount`. Continue to next document.

**Step 5d — Update document to Arhivirano (If-Match)**
Action: SharePoint HTTP PATCH
Header: `If-Match: <varItemETag>`
Fields to update (UNKNOWN internal column names):
- `Stanje` = `Arhivirano`
- `ArhivskiZnak` = document `arhivskiZnak`
- `ArchivedAt` = `utcNow()` (UNKNOWN column name)
- `ArchivedBy` = input `initiatorEmail` (UNKNOWN column name)

If 412 Precondition Failed (item changed since read):
- Re-read item (Step 5b).
- If new `Stanje` is already `Arhivirano`: treat as success (idempotent).
- If new `Stanje` is something else: add to failures with code `CONCURRENT_UPDATE`. Continue.

If other error: add to failures. Increment `varFailedCount`. Continue.

**Step 5e — Log per-document archive event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ArchiveDocument`
- EventCategory: `Archive`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: `documentItemId`
- DelovodniBroj: `delovodniBroj`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Dokument ", delovodniBroj, " je arhiviran. Arhivski znak: ", arhivskiZnak)`

Increment `varArchivedCount`.

### Finalization

**Step 6 — Determine overall success**
If `varFailedCount = 0`: overall `success = true`.
If `varFailedCount > 0`: overall `success = false`.

**Step 7 — Log overall result**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ArchiveDocument`
- Severity: `Info` if all succeeded, `Warning` if partial, `Error` if all failed
- Status: `Success` if all succeeded, `Failed` if any failed
- CorrelationId: `varCorrelationId`
- Message: `concat("Arhiviranje završeno. Uspešno: ", varArchivedCount, ", Neuspešno: ", varFailedCount)`

**Step 8 — Return response**

### Catch scope (global)

Handles unexpected errors outside the per-document loop.

Call `CF_DocCentralV3_LogEvent`:
- EventType: `ArchiveDocument`
- Severity: `Error`
- Status: `Failed`
- CorrelationId: `varCorrelationId`

Return failure response.

## Stanje transition rules

| Allowed from | Allowed to |
|---|---|
| Zavedeno | Arhivirano |

All other transitions are rejected. Specifically:
- `U odobravanju` → `Arhivirano`: NOT allowed. Document must exit the approval process first.
- `Odobreno` → `Arhivirano`: NOT allowed directly via this flow (verify against process docs — UNKNOWN whether Odobreno can be archived directly; assume not without explicit confirmation).
- `Odbijeno` → `Arhivirano`: NOT allowed.

## Error codes

| Code | Meaning |
|---|---|
| NO_DOCUMENTS_PROVIDED | Input documents array is empty |
| DOC_NOT_FOUND | Document item does not exist in Svi predmeti |
| INVALID_STATUS_FOR_ARCHIVE | Document is not in Zavedeno status |
| INVALID_ARCHIVE_SIGN | Archive sign not in App Config allowed list |
| CONCURRENT_UPDATE | Item modified by another process during archiving |
| ARCHIVE_UPDATE_FAILED | SharePoint PATCH failed for unexpected reason |

## Open items

| Item | Status |
|---|---|
| Svi predmeti Stanje internal column name | UNKNOWN |
| Svi predmeti ArhivskiZnak internal column name | UNKNOWN |
| ArchivedAt column name in Svi predmeti | UNKNOWN — may not exist yet |
| ArchivedBy column name in Svi predmeti | UNKNOWN — may not exist yet |
| App Config key for archive signs list | UNKNOWN |
| Whether Odobreno → Arhivirano is permitted | TO BE CONFIRMED — currently assumed not allowed |
| Maximum documents per call (performance limit) | UNKNOWN — suggest 50 as initial limit |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Single document, valid state and sign | Archived, audit logged |
| Multiple documents, all valid | All archived, success response |
| One document invalid status | That one fails, others succeed, partial failure response |
| Invalid archive sign | Validation fails for that document, others proceed |
| Document not found | DOC_NOT_FOUND for that item |
| Concurrent update (412) | Retry read, handle based on new Stanje |
| Already Arhivirano (concurrent) | Treated as success (idempotent) |
| Empty documents array | Immediate failure |
