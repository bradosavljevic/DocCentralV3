# Implementation steps: CF_DocCentralV3_GenerateArchiveBookPdf

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- Exports library (`EV_DocCentralV3_docExports`) exists with Write access for the flow service account.
- All Svi predmeti internal column names required for the archive book confirmed (all UNKNOWN).
- App Config key for organization name confirmed (UNKNOWN).
- Year filter strategy confirmed: dedicated year column vs date range filter (UNKNOWN).
- HTML template reviewed — see `PACode/sharepoint/archive-book-output.md`.
- Admin utility flow rules understood — see `PACode/flows/admin-utility-flow-rules.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_GenerateArchiveBookPdf`.
3. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| year | Number | `triggerBody()?['decimal_year']` |
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

`year` is Number type — uses `decimal_` prefix.

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varYear | Integer | `int(triggerBody()?['decimal_year'])` |
| varOutputFileName | String | `concat('ArhivskaKnjiga_', string(int(triggerBody()?['decimal_year'])), '_', formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), '.html')` |
| varOrgName | String | `'[Naziv organizacije]'` |
| varHtmlRows | String | `''` |
| varRowNumber | Integer | `0` |
| varTotalCount | Integer | `0` |
| varAllDocuments | Array | `[]` |
| varFileWebUrl | String | `''` |
| varNextLink | String | `''` |
| varPaginationDone | Boolean | `false` |

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `GenerateArchiveBookPdf` / eventCategory: `Archive`
- severity: `Info` / status: `Started`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Pokretanje generisanja arhivske knjige za godinu ', string(variables('varYear')), '.')`

---

## Step 5 — Try scope

### 5a — Read organization name from App Config

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter: UNKNOWN — filter by org name key (UNKNOWN column name and key value)
- Top: 1

Rename: `Get_OrgName`

Add **Condition**: `greater(length(body('Get_OrgName')?['value']), 0)`

Yes: Set `varOrgName` = `first(body('Get_OrgName')?['value'])?['UNKNOWN_OrgNameValue']`
No: leave `varOrgName` = `'[Naziv organizacije]'` (placeholder)

### 5b — Read archived documents with pagination

Archived documents for the year may exceed 5000 items in long-lived registries.
Use a Do-Until loop with `@odata.nextLink` pagination.

**Step 5b-i — First page**

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- Filter: `UNKNOWN_Stanje eq 'Arhivirano' and <year filter>`
  - Year filter Option A (dedicated column): `UNKNOWN_Year eq <varYear>`
  - Year filter Option B (date range): `UNKNOWN_FilingDate ge '2026-01-01T00:00:00Z' and UNKNOWN_FilingDate lt '2027-01-01T00:00:00Z'`
  - UNKNOWN — confirm from Svi predmeti schema
- Top: 5000
- Order by: UNKNOWN (see build notes for sort strategy)
- Select: UNKNOWN (select only the 9 archive book columns — after internal names confirmed)

Rename: `Get_ArchivedDocs_Page1`

Set `varAllDocuments` = `body('Get_ArchivedDocs_Page1')?['value']`
Set `varNextLink` = `body('Get_ArchivedDocs_Page1')?['@odata.nextLink']`

**Step 5b-ii — Pagination loop**

Add **Do Until**: `or(empty(variables('varNextLink')), variables('varPaginationDone'))`

Inside loop:

Add **SharePoint — Send an HTTP Request** — GET:
- Uri: `variables('varNextLink')` (the nextLink URL from the previous page)
- Headers: `Accept: application/json;odata=verbose`

Rename: `Get_ArchivedDocs_NextPage`

Add **Compose** — `Get_NextPage_Value`:
```
body('Get_ArchivedDocs_NextPage')?['d']?['results']
```
(The exact JSON path depends on OData format. May also be `body(...)?['value']` — test after first page.)

Add **Compose** — `Append_NextPage`:
```
union(variables('varAllDocuments'), outputs('Get_NextPage_Value'))
```

Set `varAllDocuments` = `outputs('Append_NextPage')`

Set `varNextLink` = `body('Get_ArchivedDocs_NextPage')?['@odata.nextLink']`
(or: `body(...)?['d']?['__next']` in verbose mode)

Add **Condition**: `empty(variables('varNextLink'))`
Yes: Set `varPaginationDone` = `true`

Do Until limit: 20 iterations (covers up to 100,000 documents at 5000 per page).

**Note:** If SharePoint pagination is complex to implement, and the archive book is expected to contain fewer than 5000 archived documents per year, skip the pagination loop and use Top: 5000 only. Add a comment in the implementation notes if pagination is skipped.

Set `varTotalCount` = `length(variables('varAllDocuments'))`

### 5c — Handle empty result

Add **Condition**: `equals(variables('varTotalCount'), 0)`

Yes (no archived docs):
Set `varHtmlRows` to the empty-row template:
```
concat('<tr class="empty-row"><td colspan="9">Nema arhiviranih dokumenata za godinu ', string(variables('varYear')), '.</td></tr>')
```
Skip Step 5d (Apply to each — no items to process).

No: proceed to 5d.

### 5d — Apply to each: build HTML rows

Add **Apply to each**: input = `variables('varAllDocuments')`

**Important:** Set Apply to each Concurrency to 1 (default). Do not enable parallel processing — `varHtmlRows` and `varRowNumber` are modified in each iteration.

Inside loop:

Add **Compose** — `Get_CurrentDoc`:
```
item()
```

Increment `varRowNumber`: Set `varRowNumber` = `add(variables('varRowNumber'), 1)`

#### Date formatting

Add **Compose** — `Format_FilingDate`:
```
if(
  empty(outputs('Get_CurrentDoc')?['UNKNOWN_FilingDate']),
  '',
  formatDateTime(outputs('Get_CurrentDoc')?['UNKNOWN_FilingDate'], 'dd.MM.yyyy')
)
```

Add **Compose** — `Format_ArhiviranoDate`:
```
if(
  empty(outputs('Get_CurrentDoc')?['UNKNOWN_Arhivirano']),
  '',
  formatDateTime(outputs('Get_CurrentDoc')?['UNKNOWN_Arhivirano'], 'dd.MM.yyyy')
)
```

#### HTML escaping per field

For each text field, apply HTML escaping (see build notes for the escape expression):

Add **Compose** — `Escape_DelovodniBroj`:
```
replace(replace(replace(replace(string(outputs('Get_CurrentDoc')?['UNKNOWN_DelovodniBroj']), '&', '&amp;'), '<', '&lt;'), '>', '&gt;'), '"', '&quot;')
```

Apply same pattern for: Title/Subject, PartnerNazivSnapshot, DocumentType, ArhivskiZnak, Arhivirao.

(Six separate Compose actions — one per text field. Dates do not need escaping.)

#### Build HTML row

Add **Compose** — `Build_HtmlRow`:
```
concat(
  '<tr>',
  '<td>', string(variables('varRowNumber')), '</td>',
  '<td>', outputs('Escape_DelovodniBroj'), '</td>',
  '<td>', outputs('Format_FilingDate'), '</td>',
  '<td>', outputs('Escape_Subject'), '</td>',
  '<td>', outputs('Escape_PartnerNazivSnapshot'), '</td>',
  '<td>', outputs('Escape_DocumentType'), '</td>',
  '<td>', outputs('Escape_ArhivskiZnak'), '</td>',
  '<td>', outputs('Format_ArhiviravoDate'), '</td>',
  '<td>', outputs('Escape_Arhivirao'), '</td>',
  '</tr>',
  decodeUriComponent('%0A')
)
```

Append to varHtmlRows:
```
[Compose: Append_HtmlRow]
concat(variables('varHtmlRows'), outputs('Build_HtmlRow'))

[Set variable: varHtmlRows]
outputs('Append_HtmlRow')
```

### 5e — Compose full HTML document

Add **Compose** — `Build_HtmlDocument`:

Assemble the complete HTML using the template from `PACode/sharepoint/archive-book-output.md`:

```
concat(
  uriComponentToString('%EF%BB%BF'),
  '<!DOCTYPE html><html lang="sr"><head>',
  '<meta charset="UTF-8">',
  '<meta name="viewport" content="width=device-width, initial-scale=1.0">',
  '<title>Arhivska knjiga ', string(variables('varYear')), '</title>',
  '<style>',
  '* { box-sizing: border-box; }',
  'body { font-family: Arial, sans-serif; font-size: 11pt; margin: 15mm 20mm; color: #111; }',
  'h1 { text-align: center; font-size: 13pt; margin-bottom: 4px; text-transform: uppercase; }',
  '.subtitle { text-align: center; font-size: 11pt; margin-bottom: 16px; }',
  '.meta { font-size: 9pt; margin-bottom: 20px; color: #444; }',
  '.meta span { display: inline-block; margin-right: 30px; }',
  'table { width: 100%; border-collapse: collapse; font-size: 9pt; }',
  'thead th { background-color: #e8e8e8; border: 1px solid #666; padding: 5px 6px; text-align: left; font-weight: bold; }',
  'tbody td { border: 1px solid #aaa; padding: 3px 6px; vertical-align: top; }',
  'tbody tr:nth-child(even) { background-color: #f7f7f7; }',
  '.empty-row td { text-align: center; font-style: italic; color: #666; padding: 12px; }',
  '.footer { margin-top: 20px; font-size: 8pt; color: #888; text-align: right; }',
  '@media print { body { margin: 8mm 10mm; font-size: 9pt; } thead { display: table-header-group; } tbody tr { page-break-inside: avoid; } .footer { position: fixed; bottom: 5mm; right: 10mm; } @page { size: A4 landscape; margin: 10mm; } }',
  '</style></head><body>',
  '<h1>', variables('varOrgName'), '</h1>',
  '<div class="subtitle">ARHIVSKA KNJIGA ZA ', string(variables('varYear')), '. GODINU</div>',
  '<div class="meta">',
  '<span>Datum generisanja: ', formatDateTime(utcNow(), 'dd.MM.yyyy HH:mm'), '</span>',
  '<span>Ukupan broj dokumenata: ', string(variables('varTotalCount')), '</span>',
  '</div>',
  '<table><thead><tr>',
  '<th style="width:40px">R. br.</th>',
  '<th style="width:100px">Delovodni broj</th>',
  '<th style="width:90px">Datum zavođenja</th>',
  '<th>Predmet / sadržaj</th>',
  '<th style="width:130px">Partner</th>',
  '<th style="width:100px">Tip dokumenta</th>',
  '<th style="width:80px">Arhivski znak</th>',
  '<th style="width:100px">Datum arhiviranja</th>',
  '<th style="width:120px">Ko je arhivirao</th>',
  '</tr></thead><tbody>',
  variables('varHtmlRows'),
  '</tbody></table>',
  '<div class="footer">Generisano: ', formatDateTime(utcNow(), 'dd.MM.yyyy HH:mm'), ' | DocCentralV3</div>',
  '</body></html>'
)
```

Note: `uriComponentToString('%EF%BB%BF')` prepends the UTF-8 BOM as the first character.

### 5f — Save HTML file to Exports library

Add **SharePoint — Create file**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Folder path: UNKNOWN root path of `@parameters('gpdoccen_EV_DocCentralV3_docExports')`
- File name: `variables('varOutputFileName')`
- File content: `outputs('Build_HtmlDocument')`

Rename: `Create_HtmlFile`

Set `varFileWebUrl` from Create file response (UNKNOWN property — see ExportAppConfig build notes for pattern).

### 5g — Log Success

Call CF_DocCentralV3_LogEvent:
- eventType: `GenerateArchiveBookPdf` / severity: `Info` / status: `Success`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Arhivska knjiga za ', string(variables('varYear')), ' generisana: ', variables('varOutputFileName'), '. Ukupno dokumenata: ', string(variables('varTotalCount')), '.')`

### 5h — Respond success

**Respond to a PowerApp or flow** with 6 outputs:

| Output | Value |
|---|---|
| success | `true` |
| message | `Arhivska knjiga je uspešno generisana.` |
| fileUrl | `variables('varFileWebUrl')` |
| fileName | `variables('varOutputFileName')` |
| correlationId | `variables('varCorrelationId')` |
| errorCode | `''` |

---

## Step 6 — Catch scope

Configure Run after: failed, timed out.

Call CF_DocCentralV3_LogEvent:
- eventType: `GenerateArchiveBookPdf` / severity: `Error` / status: `Failed`
- errorCode: `ARCHIVE_BOOK_GENERATION_FAILED`
- errorMessage: `result('Try')?[0]?['error']?['message']`
- correlationId: `variables('varCorrelationId')`

Respond failure:
- success: false
- message: `Greška pri generisanju arhivske knjige.`
- errorCode: `ARCHIVE_BOOK_GENERATION_FAILED`
- fileUrl: `''`
- fileName: `''`
- correlationId: `variables('varCorrelationId')`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_GenerateArchiveBookPdf`
- [ ] Trigger: Power Apps (V2) with 3 inputs (year as Number)
- [ ] 11 variables initialized; varYear = int(decimal_year)
- [ ] Log Started before Try scope
- [ ] App Config org name read (fallback to placeholder if not found)
- [ ] Get_ArchivedDocs_Page1 filters by Stanje = Arhivirano AND year
- [ ] Pagination loop handles @odata.nextLink (or documented skip for < 5000 expectation)
- [ ] Empty result: single empty-data row rendered — no error
- [ ] Apply to each: concurrency = 1 (default, not parallel)
- [ ] Date values formatted as dd.MM.yyyy (null-safe)
- [ ] All text values HTML-escaped before insertion
- [ ] UTF-8 BOM prepended as first character of HTML
- [ ] Full HTML assembled from template (see archive-book-output.md)
- [ ] Create file: Exports library root, .html extension
- [ ] varFileWebUrl set from Create file response
- [ ] Log Success with document count in message
- [ ] Catch scope: log Error, respond failure
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow inside DocCentralV3 solution
