# Flow design: CF_DocCentralV3_CreateDocument

## Purpose

Central orchestration flow for document registration.
This flow receives the full document registration payload from the Canvas App, coordinates all child flows and SharePoint operations, and returns a structured response.

This is the most complex flow in the solution. All write operations for a new document originate here.

## Trigger type

Power Apps V2. Called directly from the Canvas App when the user submits the document registration form.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Office365Users` (for user display name resolution if needed)

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`
- `gpdoccen_EV_DocCentralV3_docDokumenti`

## Child flows called

- `CF_DocCentralV3_LogEvent`
- `CF_DocCentralV3_GenerateRegistryNumber`
- `CF_DocCentralV3_UseReservedNumber`
- `CF_DocCentralV3_AssignPermissions`

## Input schema

```json
{
  "documentType": "",
  "metadata": {
    "partnerNaziv": "",
    "partnerId": null,
    "partnerPIBSnapshot": "",
    "partnerMestoSnapshot": "",
    "partnerAdresaSnapshot": "",
    "subject": "",
    "description": "",
    "customFields": {}
  },
  "useReservedNumber": false,
  "reservedNumberId": null,
  "filingDate": "",
  "mainFile": {
    "fileName": "",
    "fileContentBase64": ""
  },
  "attachments": [
    {
      "fileName": "",
      "fileContentBase64": ""
    }
  ],
  "initiatorEmail": "",
  "correlationId": ""
}
```

Notes:
- `mainFile` is required. If not provided, flow fails validation.
- `attachments` may be an empty array.
- `useReservedNumber` determines which number acquisition path is used.
- `reservedNumberId` is required if `useReservedNumber = true`.
- `filingDate` is required if `useReservedNumber = true`. For new numbers, it is set to today by the flow.
- `metadata.customFields` holds any document-type-specific fields. Exact keys UNKNOWN until App Config schema is confirmed.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Dokument je uspešno zaveden.",
  "itemId": 0,
  "delovodniBroj": "",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Dokument nije zaveden.",
  "errorCode": "",
  "correlationId": ""
}
```

## Flow steps

### Initialization

**Step 1 — Initialize variables**
- `varCorrelationId` = input `correlationId` if not empty, else `guid()`
- `varDelovodniBroj` (String) = empty
- `varSviPredmetiItemId` (Integer) = 0
- `varMainFileSystemName` (String) = empty
- `varFilingDate` (String) = empty
- `varUsedReservedNumber` (Boolean) = false

**Step 2 — Log Started event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `CreateDocument`
- EventCategory: `Document`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje zavođenja dokumenta."

### Validation (Try scope begins here)

**Step 3 — Validate required inputs**
Check:
- `documentType` is not empty.
- `mainFile.fileName` is not empty.
- `mainFile.fileContentBase64` is not empty.
- If `useReservedNumber = true`: `reservedNumberId` is not null and `filingDate` is not empty.
- `initiatorEmail` is not empty.

If any check fails:
- Log validation failure via `CF_DocCentralV3_LogEvent` (Severity: Warning, EventType: ValidationError, Status: Failed).
- Return failure response with code `VALIDATION_FAILED` and a specific message describing the missing field.
- Exit flow.

**Step 4 — Validate document type against App Config**
Read document types from App Config (UNKNOWN key).
Check that input `documentType` is in the allowed list.
If not found: log warning. Return failure with code `INVALID_DOCUMENT_TYPE`.

**Step 5 — Validate active year / registry book**
Read active year from App Config.
If no active registry book exists or the year is closed: log error. Return failure with code `REGISTRY_YEAR_CLOSED`.

### Number acquisition

**Step 6 — Acquire DelovodniBroj**

Branch on `useReservedNumber`:

**If `useReservedNumber = false` (generate new number):**
- Call child flow `CF_DocCentralV3_GenerateRegistryNumber`.
- Input: `initiatorEmail`, `correlationId: varCorrelationId`.
- If response `success = false`: propagate failure. Return failure response. Exit flow.
- Store: `varDelovodniBroj` = response `delovodniBroj`.
- Set `varFilingDate` = today's date (`utcNow('yyyy-MM-dd')`).

**If `useReservedNumber = true` (use reserved number):**
- Call child flow `CF_DocCentralV3_UseReservedNumber`.
- Input: `reservedNumberId`, `requestedYear` (from active year in App Config), `initiatorEmail`, `correlationId: varCorrelationId`.
- If response `success = false`: propagate failure. Return failure response. Exit flow.
- Store: `varDelovodniBroj` = response `delovodniBroj`.
- Set `varFilingDate` = input `filingDate`.
- Set `varUsedReservedNumber = true`.
- Store: `varReservedNumberItemId` = input `reservedNumberId` (for deletion after successful creation).

### Create Svi predmeti item

**Step 7 — Build SafeDelovodniBroj**
Remove characters not safe for file names from `varDelovodniBroj`.
Replace `/` with `-`. Remove spaces, special chars.
Store as `varSafeDelovodniBroj`.

**Step 8 — Create item in Svi predmeti**
Action: SharePoint — Create item
List: `EV_DocCentralV3_lstSviPredmeti`

Mapped fields (internal column names UNKNOWN for unlisted fields — use documented names where available):

| Column | Value | Internal name status |
|---|---|---|
| Title | input `metadata.subject` | Standard |
| DelovodniBroj | `varDelovodniBroj` | UNKNOWN |
| Stanje | `Zavedeno` | UNKNOWN |
| DocumentType | input `documentType` | UNKNOWN |
| FilingDate | `varFilingDate` | UNKNOWN |
| InitiatorEmail | input `initiatorEmail` | UNKNOWN |
| PartnerId | input `metadata.partnerId` | UNKNOWN |
| PartnerNazivSnapshot | input `metadata.partnerNaziv` | UNKNOWN |
| PartnerPIBSnapshot | input `metadata.partnerPIBSnapshot` | UNKNOWN |
| PartnerMestoSnapshot | input `metadata.partnerMestoSnapshot` | UNKNOWN |
| PartnerAdresaSnapshot | input `metadata.partnerAdresaSnapshot` | UNKNOWN |
| CorrelationId | `varCorrelationId` | UNKNOWN |

Store: `varSviPredmetiItemId` = ID of the created item.

If creation fails: log error. Return failure. Exit flow. (Number was already generated — this is logged as a Failed audit event with the generated number for manual reconciliation.)

### Create main document file

**Step 9 — Generate unique system file name for main document**
Expression:
```
concat(varSafeDelovodniBroj, '_', guid(), '_', sanitize(mainFile.fileName))
```
`sanitize()` = remove/replace characters not safe for SharePoint file names.
Store as `varMainFileSystemName`.

**Step 10 — Create main file in document library root**
Action: SharePoint — Create file
Site: `EV_DocCentralV3_SharePointSite`
Folder path: `/` + value of `EV_DocCentralV3_docDokumenti` (root of library — no subfolder).
File name: `varMainFileSystemName`
File content: decoded from `mainFile.fileContentBase64`

Store: `varMainFileId` = ID of the created file item.

If creation fails: log error. Return failure. Exit flow. (Svi predmeti item was already created — logged as partial failure for manual reconciliation. Consider marking Stanje as FAILED_PARTIAL in a cleanup step.)

**Step 11 — Update main file metadata**
Action: SharePoint — Update file properties
Site: `EV_DocCentralV3_SharePointSite`
Library: `EV_DocCentralV3_docDokumenti`
Item ID: `varMainFileId`

Fields:

| Column | Value | Internal name status |
|---|---|---|
| DelovodniBroj | `varDelovodniBroj` | UNKNOWN |
| SviPredmetiId | `varSviPredmetiItemId` | UNKNOWN |
| IsPrilog | false | UNKNOWN |
| OriginalFileName | input `mainFile.fileName` | UNKNOWN |
| SystemFileName | `varMainFileSystemName` | UNKNOWN |
| DocumentType | input `documentType` | UNKNOWN |
| DocumentStatus | `Zavedeno` | UNKNOWN |
| CreatedByApp | true | UNKNOWN |
| CorrelationId | `varCorrelationId` | UNKNOWN |

### Create attachments

**Step 12 — Apply for each attachment (if any)**
Apply to each item in `attachments` array:

**Step 12a — Generate unique system file name for attachment**
```
concat(varSafeDelovodniBroj, '_PRILOG_', guid(), '_', sanitize(attachment.fileName))
```
Store as `varAttachmentSystemName`.

**Step 12b — Create attachment file in library root**
Action: SharePoint — Create file
Folder path: root of `EV_DocCentralV3_docDokumenti`.
File name: `varAttachmentSystemName`
File content: decoded from `attachment.fileContentBase64`

Store: `varAttachmentFileId` = ID of created file item.

**Step 12c — Update attachment metadata**
Action: SharePoint — Update file properties

| Column | Value |
|---|---|
| DelovodniBroj | `varDelovodniBroj` |
| SviPredmetiId | `varSviPredmetiItemId` |
| IsPrilog | true |
| ParentDelovodniBroj | `varDelovodniBroj` |
| ParentDocumentId | `varSviPredmetiItemId` |
| OriginalFileName | `attachment.fileName` |
| SystemFileName | `varAttachmentSystemName` |
| DocumentType | input `documentType` |
| DocumentStatus | `Zavedeno` |
| CreatedByApp | true |
| CorrelationId | `varCorrelationId` |

### Permission assignment

**Step 13 — Assign permissions**
Call child flow `CF_DocCentralV3_AssignPermissions`:
- Input: `varSviPredmetiItemId`, `initiatorEmail`, document type, active org unit (UNKNOWN — read from App Config), `varCorrelationId`.
- If permissions fail: log warning. Do NOT fail the document creation. Document is considered created. Permissions failure is logged and may be retried.

### Post-creation cleanup

**Step 14 — Delete reserved number (if applicable)**
Condition: `varUsedReservedNumber = true`

Action: SharePoint — Delete item
List: `EV_DocCentralV3_lstRezervisaniBrojevi`
ID: `varReservedNumberItemId`

If delete fails: log warning. Do not fail document creation. Document is already created. Orphaned reserved number is flagged in audit log for manual cleanup.

After deletion: log event:
- EventType: `UseReservedNumber`
- Status: `Success`
- Message: reserved number consumed.

### Final audit and response

**Step 15 — Log Success event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `CreateDocument`
- EventCategory: `Document`
- Severity: `Info`
- Status: `Success`
- DocumentItemId: `varSviPredmetiItemId`
- DelovodniBroj: `varDelovodniBroj`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Dokument ", varDelovodniBroj, " je uspešno zaveden.")`

**Step 16 — Return success response**
```json
{
  "success": true,
  "message": "Dokument je uspešno zaveden.",
  "itemId": "<varSviPredmetiItemId>",
  "delovodniBroj": "<varDelovodniBroj>",
  "correlationId": "<varCorrelationId>"
}
```

## Error handling (Catch scope)

Catch scope wraps Steps 3–15.

In the Catch scope:
- Compose error details from `result()` of failed actions.
- Call `CF_DocCentralV3_LogEvent`:
  - EventType: `CreateDocument`
  - Severity: `Error`
  - Status: `Failed`
  - ErrorCode: `CREATE_DOCUMENT_FAILED`
  - ErrorMessage: composed error details
  - CorrelationId: `varCorrelationId`
- Return failure response:
```json
{
  "success": false,
  "message": "Dokument nije zaveden. Proverite audit log za detalje.",
  "errorCode": "CREATE_DOCUMENT_FAILED",
  "correlationId": "<varCorrelationId>"
}
```

## Partial failure handling

| Failure point | Document state | Action |
|---|---|---|
| Validation fails | Nothing created | Return error. Clean state. |
| Number generation fails | Nothing created | Return error. Clean state. |
| Svi predmeti item creation fails | Number generated but item not created | Log as Failed with the generated number for reconciliation. |
| Main file creation fails | Svi predmeti item exists, file missing | Log as Failed. Mark item Stanje = FAILED_PARTIAL if possible. |
| Attachment creation fails | Document partially created | Log warning. Continue. Document is still considered valid; attachment failure is non-fatal. |
| Permissions assignment fails | Document created, permissions missing | Log warning. Non-fatal. Flag for retry. |
| Reserved number deletion fails | Document created, reserved number orphaned | Log warning. Non-fatal. Manual cleanup required. |

## Error codes

| Code | Meaning |
|---|---|
| VALIDATION_FAILED | Required input missing or invalid |
| INVALID_DOCUMENT_TYPE | DocumentType not in App Config allowed list |
| REGISTRY_YEAR_CLOSED | Active registry year is closed |
| REGISTRY_NUMBER_GENERATION_FAILED | Child flow GenerateRegistryNumber failed |
| RESERVED_NUMBER_INVALID | Child flow UseReservedNumber failed |
| CREATE_ITEM_FAILED | Svi predmeti item creation failed |
| CREATE_FILE_FAILED | Main file creation in library failed |
| CREATE_DOCUMENT_FAILED | General unhandled failure |

## File name sanitization rules

Characters to remove or replace in file names before creating in SharePoint:

- Replace `/` with `-`
- Replace `\` with `-`
- Replace `:` with `-`
- Remove `*`, `?`, `"`, `<`, `>`, `|`
- Replace multiple spaces with single space
- Trim leading and trailing spaces and dots

## Open items

| Item | Status |
|---|---|
| Svi predmeti internal column names | UNKNOWN — require list schema confirmation |
| Document library internal column names | UNKNOWN — require library schema confirmation |
| customFields mapping to Svi predmeti columns | UNKNOWN — depends on AppConfig.csv and document types |
| Org unit / permission group source in App Config | UNKNOWN |
| FAILED_PARTIAL status handling | To be confirmed — is this a valid status in App Config |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Valid new document, new number | Document created, number generated, file in root, success response |
| Valid document with reserved number | Reserved number consumed and deleted, document created |
| Valid document with attachments | Main file and all attachments in root, all linked by metadata |
| Missing mainFile | Validation fails, nothing created, error response |
| Invalid documentType | Validation fails, nothing created, error response |
| Number generation conflict | Retry in child flow, eventual success or failure propagated |
| Reserved number already used | Failure propagated, nothing created |
| File with same original name as existing | No overwrite — system name is unique due to guid() |
| Two users submit simultaneously | Each gets unique DelovodniBroj, no race condition |
| SharePoint write fails mid-flow | Partial failure logged, error response returned |
