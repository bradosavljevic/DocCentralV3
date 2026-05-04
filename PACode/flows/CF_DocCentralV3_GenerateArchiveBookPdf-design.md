# Flow design: CF_DocCentralV3_GenerateArchiveBookPdf

## Purpose

Generates an HTML archive book (`Arhivska knjiga`) for a specified registry year and saves
it to the `Exports` SharePoint library root.

The HTML file is print-ready / PDF-ready. The final PDF is produced by the user via
browser print → Save as PDF. No premium connectors, no Word templates, no OneDrive
conversion, no third-party connectors are required.

Scope for v1: HTML file only. True PDF connector-based generation is out of scope.

## Trigger type

Power Apps V2. Called from Canvas App (Administracija screen or Arhiviranje screen).

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

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

- `year`: the registry year for which to generate the archive book. May be the active year
  or a previously closed year. No restriction on year value.

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

## Confirmed Svi predmeti fields used in archive book

| Display name | Confirmed | Internal name | Notes |
|---|---|---|---|
| DelovodniBroj | Confirmed | UNKNOWN | Registry number |
| Stanje | Confirmed | UNKNOWN | Filter: must be Arhivirano |
| Arhivirano | Confirmed | UNKNOWN | Date of archiving |
| Arhivirao | Confirmed | UNKNOWN | User who archived |
| PartnerNazivSnapshot | Confirmed | UNKNOWN | Historical partner name |

Additional fields for archive book content (display names assumed, UNKNOWN internal names):

| Display name (assumed) | Internal name | Notes |
|---|---|---|
| Title / Subject | UNKNOWN | Document subject/content description |
| FilingDate | UNKNOWN | Date of filing — may be stored as date or derived from DelovodniBroj year |
| DocumentType | UNKNOWN | Type of document |
| ArhivskiZnak | UNKNOWN | Archive sign/classification code |

All internal column names remain UNKNOWN until Svi predmeti schema is confirmed.

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varOutputFileName` = `concat("ArhivskaKnjiga_", string(year), "_", formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), ".html")`
- `varHtmlRows` (String) = empty
- `varRowNumber` (Integer) = 0

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- EventCategory: `Archive`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Pokretanje generisanja arhivske knjige za godinu ", string(year), ".")`

**Step 3 — Read organization name from App Config**
Action: SharePoint — Get items from App Config
Filter: UNKNOWN key for organization display name.
Store as `varOrgName`.
If not found or empty: use placeholder `[Naziv organizacije]`.

**Step 4 — Read archived documents for the year (with pagination)**
Action: SharePoint — Get items (Do-Until with `@odata.nextLink` pagination)
List: `EV_DocCentralV3_lstSviPredmeti`
Filter: `Stanje = 'Arhivirano'` AND year = input `year`

Year filter approach (UNKNOWN — choose one based on actual schema):
- Option A: filter by a dedicated year column if it exists (UNKNOWN column name).
- Option B: filter by FilingDate range: `FilingDate ge '2026-01-01' and FilingDate lt '2027-01-01'`.
- Option C: filter only by `Stanje = 'Arhivirano'` and apply year filter in memory if delegation is not possible.

Prefer server-side filtering (Options A or B) for delegability and performance.

Order: by `DelovodniBroj` ascending using numeric sort if a separate counter column exists;
otherwise by a sortable date or ID column.

Collect all results across pages into `varDocuments`.
Store total count as `varTotalCount`.

**Step 5 — Build HTML row content**
Apply to each document in `varDocuments`:
- Increment `varRowNumber` by 1.
- Format `Arhivirano` date as `dd.MM.yyyy` (or empty string if null).
- Format `FilingDate` as `dd.MM.yyyy` (or empty string if null).
- Escape HTML special characters in text values (`<`, `>`, `&`, `"`) to prevent HTML injection.
- Append one HTML `<tr>` row to `varHtmlRows`.

Row template:
```html
<tr>
  <td>{rowNumber}</td>
  <td>{DelovodniBroj}</td>
  <td>{FilingDate formatted}</td>
  <td>{Subject / Title}</td>
  <td>{PartnerNazivSnapshot}</td>
  <td>{DocumentType}</td>
  <td>{ArhivskiZnak}</td>
  <td>{Arhivirano formatted}</td>
  <td>{Arhivirao}</td>
</tr>
```

If `varDocuments` is empty: `varHtmlRows` = a single row spanning all columns with message
`"Nema arhiviranih dokumenata za ovu godinu."`.

**Step 6 — Compose full HTML document**
Compose the complete HTML string by combining:
- HTML header and metadata (see HTML structure in `archive-book-output.md`)
- Inline CSS for print-friendly layout
- Table headers
- `varHtmlRows`
- HTML footer with generation metadata

Store as `varHtmlContent`.

Full HTML structure — see `PACode/sharepoint/archive-book-output.md` for the template.

Key CSS for print:
```css
@media print {
  body { margin: 10mm; font-size: 9pt; }
  thead { display: table-header-group; }
  tr { page-break-inside: avoid; }
  @page { size: A4 landscape; margin: 10mm; }
}
```

The `@page` rule sets A4 landscape orientation, which is appropriate for a wide table.

**Step 7 — Save HTML file to Exports library root**
Action: SharePoint — Create file
Site: `EV_DocCentralV3_SharePointSite`
Folder path: root of `EV_DocCentralV3_docExports` (no subfolder)
File name: `varOutputFileName`
File content: `varHtmlContent` encoded as UTF-8

UTF-8 BOM (`﻿` — character U+FEFF) must be prepended to `varHtmlContent` before saving.
This ensures Serbian characters (č, ć, š, ž, đ) render correctly when opened in Windows browsers.

Implementation note: In Power Automate, UTF-8 BOM can be prepended using:
`concat(uriComponentToString('%EF%BB%BF'), varHtmlContent)`

Store `varFileWebUrl` from the create file response (SharePoint file URL).

**Step 8 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- EventCategory: `Archive`
- Severity: `Info`
- Status: `Success`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Arhivska knjiga za ", string(year), " generisana: ", varOutputFileName, ". Ukupno dokumenata: ", string(varTotalCount), ".")`

**Step 9 — Return success response**
Return `fileUrl` = `varFileWebUrl` and `fileName` = `varOutputFileName`.

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateArchiveBookPdf`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `ARCHIVE_BOOK_GENERATION_FAILED`
- CorrelationId: `varCorrelationId`

Return failure response.

## HTML encoding

- File encoding: UTF-8 with BOM.
- HTML `<meta charset="UTF-8">` is always present.
- All text values from SharePoint are HTML-escaped before insertion into the template.
- Date values are formatted as `dd.MM.yyyy` using Power Automate `formatDateTime()`.

## File naming

Format: `ArhivskaKnjiga_{year}_{YYYYMMDD_HHmmss}.html`
Example: `ArhivskaKnjiga_2026_20261215_083012.html`

Files are saved in the root of the Exports library. No subfolders.
No automatic cleanup. Old export files are deleted manually by admins.

## Error codes

| Code | Meaning |
|---|---|
| ARCHIVE_BOOK_GENERATION_FAILED | General failure during read or file creation |
| FILE_SAVE_FAILED | Could not save HTML file to Exports library |

Note: Empty result (no archived documents for the year) is not an error. The file is
created with headers and an empty-data message row, and success is returned.

## Open items

| Item | Status |
|---|---|
| Svi predmeti internal column names for all archive book fields | UNKNOWN |
| Year filter strategy (dedicated year column vs date range) | UNKNOWN — depends on schema |
| Organization name App Config key | UNKNOWN |
| Sort strategy for DelovodniBroj (text vs numeric) | UNKNOWN — depends on schema |
| FilingDate column internal name and type | UNKNOWN |
| DocumentType internal column name | UNKNOWN |
| ArhivskiZnak internal column name | UNKNOWN |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Year with 50 archived documents | HTML file with 50 rows, download URL returned |
| Year with no archived documents | HTML file with header and empty message, success returned |
| Year is closed prior year | Works same — no restriction on year value |
| File save to Exports library fails | Error returned, audit logged |
| Document count exceeds 5000 | Pagination collects all pages before composing HTML |
| Serbian characters in document titles | Correctly rendered due to UTF-8 BOM |
