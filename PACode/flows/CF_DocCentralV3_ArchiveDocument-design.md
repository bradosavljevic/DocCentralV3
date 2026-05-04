# Flow design: CF_DocCentralV3_ArchiveDocument

## Purpose

Archives one or more documents by transitioning their `Stanje` from `Zavedeno` to `Arhivirano`.
Applies the selected archive sign (`ArhivskiZnak`) and records the archiving date and user
on each document item in `Svi predmeti`.
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

## Confirmed Svi predmeti fields for archiving

| Display name | Type | Purpose | Internal name |
|---|---|---|---|
| Arhivirano | Date | Date the document was archived | UNKNOWN |
| Arhivirao | Single line of text | User who archived the document | UNKNOWN |

These field display names are confirmed. Internal column names are UNKNOWN until the list
schema is inspected or the columns are confirmed by the client.

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
- `arhivskiZnak`: archive classification code — validated against App Config allowed values (UNKNOWN key for archive signs list).
- `initiatorEmail`: user performing the archiving action — written to the `Arhivirao` field.

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

Partial or full failure:
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

`success = false` if any document fails, even if others succeeded.
The Canvas App uses the `failures` array to display per-document errors.

## Pre-conditions per document

Each document is validated individually before any update is made.

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
Store as `varValidArchiveSigns` (array of allowed values).

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

Each document is processed within a scoped error handler (not the global Catch).
Failures are accumulated in `varFailures`; processing continues for remaining documents.

**Step 5a — Validate archive sign**
Check document `arhivskiZnak` is in `varValidArchiveSigns`.
If not: add to `varFailures` (INVALID_ARCHIVE_SIGN). Increment `varFailedCount`. Continue.

**Step 5b — Read document item from Svi predmeti**
Action: SharePoint — Get item
ID: `documentItemId`

If not found: add to `varFailures` (DOC_NOT_FOUND). Increment `varFailedCount`. Continue.

Store:
- `varCurrentStanje` = `Stanje` value
- `varItemETag` = item `@odata.etag`

**Step 5c — Validate current Stanje**
If `varCurrentStanje` ≠ `Zavedeno`:
Add to `varFailures` (INVALID_STATUS_FOR_ARCHIVE, include actual Stanje in message).
Increment `varFailedCount`. Continue.

**Step 5d — Update document to Arhivirano (If-Match)**
Action: SharePoint HTTP PATCH
Header: `If-Match: <varItemETag>`

Fields to update (internal names UNKNOWN):
- `Stanje` → `Arhivirano`
- `ArhivskiZnak` → document `arhivskiZnak` (UNKNOWN internal name for this column)
- `Arhivirano` → `utcNow()` formatted as date (confirmed display name; internal name UNKNOWN)
- `Arhivirao` → input `initiatorEmail` (confirmed display name; internal name UNKNOWN)

If 412 Precondition Failed (concurrent update):
- Re-read item.
- If new `Stanje` is already `Arhivirano`: treat as success (idempotent — same intent).
- If new `Stanje` is something else: add to `varFailures` (CONCURRENT_UPDATE). Continue.

If other error: add to `varFailures` (ARCHIVE_UPDATE_FAILED). Increment `varFailedCount`. Continue.

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
- Message: `concat("Dokument ", delovodniBroj, " je arhiviran. Arhivski znak: ", arhivskiZnak, ".")`

Increment `varArchivedCount`.

### Finalization

**Step 6 — Determine overall success**
`success = (varFailedCount = 0)`

**Step 7 — Log overall result**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `ArchiveDocument`
- Severity: `Info` if all succeeded; `Warning` if partial; `Error` if all failed
- Status: `Success` if all succeeded; `Failed` if any failed
- CorrelationId: `varCorrelationId`
- Message: `concat("Arhiviranje završeno. Uspešno: ", varArchivedCount, ", Neuspešno: ", varFailedCount, ".")`

**Step 8 — Return response**

### Catch scope (global)

Handles unexpected errors that escape the per-document scope.

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

Other transitions explicitly rejected by this flow:
- `U odobravanju` → `Arhivirano`: NOT allowed.
- `Odobreno` → `Arhivirano`: NOT allowed via this flow (requires process confirmation — UNKNOWN).
- `Odbijeno` → `Arhivirano`: NOT allowed.

## Error codes

| Code | Meaning |
|---|---|
| NO_DOCUMENTS_PROVIDED | Input documents array is empty |
| DOC_NOT_FOUND | Document item does not exist in Svi predmeti |
| INVALID_STATUS_FOR_ARCHIVE | Document is not in Zavedeno status |
| INVALID_ARCHIVE_SIGN | Archive sign value not in App Config allowed list |
| CONCURRENT_UPDATE | Item modified by another process during archiving |
| ARCHIVE_UPDATE_FAILED | SharePoint PATCH failed for unexpected reason |

## Open items

| Item | Status |
|---|---|
| Svi predmeti Stanje internal column name | UNKNOWN |
| Svi predmeti ArhivskiZnak internal column name | UNKNOWN |
| Svi predmeti Arhivirano internal column name | UNKNOWN — display name confirmed |
| Svi predmeti Arhivirao internal column name | UNKNOWN — display name confirmed |
| App Config key for archive signs list | UNKNOWN |
| Whether Odobreno → Arhivirano is permitted | TO BE CONFIRMED — currently not allowed |
| Maximum documents per call (performance) | UNKNOWN — suggest max 50 per call |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Single document, valid state and sign | Archived; Arhivirano = today; Arhivirao = initiatorEmail; audit logged |
| Multiple documents, all valid | All archived, success response |
| One document invalid status | That one fails, others succeed, partial failure response |
| Invalid archive sign | Fails validation for that document, others proceed |
| Document not found | DOC_NOT_FOUND for that item |
| Concurrent update (412) | Re-read; if already Arhivirano treat as success; else CONCURRENT_UPDATE |
| Empty documents array | Immediate failure NO_DOCUMENTS_PROVIDED |
