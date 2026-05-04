# Flow design: CF_DocCentralV3_GenerateArchiveBookPdf

## Purpose

Generates a PDF archive book (`Arhivska knjiga`) for a specified registry year.
The archive book is a structured listing of all archived documents for that year,
including their metadata and archive signs.

Scope for v1: archive book PDF only. No other document PDFs are generated in this version.

## Trigger type

Power Apps V2. Called from Canvas App (Administracija screen or Arhiviranje screen).

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_OneDrive`

## Design note on PDF generation

Power Automate does not natively generate PDF files.
Available approaches in Power Platform context:

**Option A — HTML → PDF via OneDrive/Word template**
Generate an HTML or Word document, then convert to PDF using:
- SharePoint/OneDrive file conversion (Convert file action — converts Word .docx to PDF).

Recommended flow:
1. Compose HTML content for the archive book.
2. Convert HTML to a Word .docx using a template approach — LIMITATION: Power Automate
   cannot natively convert HTML to .docx. This approach requires a Word template with
   content controls populated by the flow.
3. Use OneDrive — Convert file action to convert .docx to PDF.
4. Save PDF to Exports library.

**Option B — HTML file (interim)**
For v1, generate an HTML file rather than a true PDF if PDF conversion tooling is unavailable.
Save the HTML file in the Exports library. The user opens it in a browser and prints to PDF.
This is a pragmatic v1 choice pending PDF tooling confirmation.

**Option C — Power Apps HTML text control + print**
Generate the archive book content in Canvas App as a rich HTML text view, allowing the user
to print from the browser. No flow-generated PDF.

**Recommended approach for v1: Option A (Word template → PDF via OneDrive conversion)**
A Word template file is pre-placed in the Exports or a templates sub-folder.
The flow copies the template, populates it via SharePoint file properties or via
the Word Online connector (UNKNOWN — availability depends on license/connector access),
then converts to PDF.

If Word Online connector is unavailable: fall back to **Option B (HTML file)**.

**Final approach: UNKNOWN — requires confirmation of available connectors and license.**

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_docExports`

## Input schema

```json
{
  "year": 2026,
  "initiatorEmail": "",
  "correlationId": ""
}
```

- `year`: the registry year for which to generate the archive book.
  May be a closed year (to generate archive book after year closing) or the active year.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Arhivska knjiga je uspešno generisana.",
  "fileUrl": "",
  "fileName": "",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Greška pri generisanju arhivske knjige.",
  "errorCode": "",
  "correlationId": ""
}
```

## Archive book content requirements

The archive book must include for each archived document (UNKNOWN exact regulatory format —
to be confirmed against Serbian archiving regulations or client requirement):

| # | Column | Source |
|---|---|---|
| 1 | Redni broj | Sequential row number |
| 2 | Delovodni broj | Svi predmeti DelovodniBroj |
| 3 | Datum zavođenja | Svi predmeti FilingDate (UNKNOWN column name) |
| 4 | Naziv predmeta / sadržaj | Svi predmeti Title or Subject (UNKNOWN) |
| 5 | Partner | PartnerNazivSnapshot from Svi predmeti |
| 6 | Tip dokumenta | DocumentType from Svi predmeti |
| 7 | Arhivski znak | ArhivskiZnak from Svi predmeti |
| 8 | Datum arhiviranja | ArchivedAt from Svi predmeti (UNKNOWN column name) |
| 9 | Ko je arhivirao | ArchivedBy from Svi predmeti (UNKNOWN column name) |

All column names in Svi predmeti are UNKNOWN until list schema is confirmed.

The archive book header must include:
- Organization name (UNKNOWN — read from App Config or hardcoded per client)
- Year
- Generation date
- Total number of documents

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varOutputFileName` = `concat("ArhivskaKnjiga_", string(year), "_", formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'))`

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- EventCategory: `Archive`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Pokretanje generisanja arhivske knjige za godinu ", string(year))`

**Step 3 — Read archived documents for the year**
Action: SharePoint — Get items (with paging)
List: `EV_DocCentralV3_lstSviPredmeti`
Filter: year = input `year` AND `Stanje = 'Arhivirano'`
Order: by `DelovodniBroj` ascending (UNKNOWN column name for ordering)
Top: up to 5000; use pagination for larger sets.

(Year filter uses UNKNOWN column name — likely `FilingYear` or extracted from `FilingDate`.)

Store all items in `varDocuments` array.

If no archived documents: log Info. Return success with an empty archive book file
(contains headers only — still a valid output).

**Step 4 — Read organization name from App Config**
Read organization display name from App Config (UNKNOWN key).
Store as `varOrgName`.
If UNKNOWN: use placeholder `[Naziv organizacije]` in the output.

**Step 5 — Compose archive book content**

Based on chosen approach (UNKNOWN — pending connector confirmation):

**Sub-option A: HTML generation**
Compose an HTML string with:
- Document header (org name, year, generation date, total count)
- HTML table with one row per document
- Styling inline (basic table borders, readable font)

Store as `varHtmlContent`.

**Sub-option B: Word template population**
(UNKNOWN — requires Word Online connector and pre-existing template)
- Copy Word template from templates location.
- Populate content controls or use Find & Replace for placeholders.
- Generate rows via template logic.

**Step 6 — Save output file to Exports library**

For HTML approach:
Action: SharePoint — Create file
File name: `varOutputFileName` + `.html`
Content: `varHtmlContent`

For PDF approach (Word → PDF):
Action: OneDrive — Convert file (Word .docx → PDF)
Save PDF to Exports library.

Store `varFileWebUrl` from the create/convert response.

**Step 7 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- Status: `Success`
- Message: `concat("Arhivska knjiga za ", string(year), " generisana: ", varOutputFileName)`

**Step 8 — Return success response**

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `ARCHIVE_BOOK_GENERATION_FAILED`
- CorrelationId: `varCorrelationId`

Return failure response.

## Error codes

| Code | Meaning |
|---|---|
| ARCHIVE_BOOK_GENERATION_FAILED | General failure |
| NO_ARCHIVED_DOCUMENTS | No archived documents found for the given year (soft — returns file with headers) |
| FILE_SAVE_FAILED | Could not save output file to Exports library |

## Open items

| Item | Status |
|---|---|
| PDF generation approach (HTML / Word+PDF / HTML only) | UNKNOWN — requires connector confirmation |
| Regulatory format of archive book | UNKNOWN — requires client/legal input |
| Svi predmeti column names for archive book content | UNKNOWN |
| AuditLog EventType for GenerateArchiveBookPdf | Present in EventType list — confirmed |
| Organization name App Config key | UNKNOWN |
| Word template file location and structure | UNKNOWN — needed if Option A chosen |
| Paging strategy for very large document sets (> 5000) | To be implemented with do-until pagination |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Year with 50 archived documents | Archive book generated with 50 rows |
| Year with no archived documents | File created with headers only, success response |
| Year input for a closed year | Works same as active year — no restriction |
| File save to Exports library fails | Error returned, audit logged |
| Document count exceeds 5000 | Pagination handles all documents |
