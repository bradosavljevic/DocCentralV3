# Implementation steps: CF_DocCentralV3_ExportAppConfig

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- Exports library (`EV_DocCentralV3_docExports`) exists and the flow's service account has Write access.
- App Config internal column names confirmed (all UNKNOWN — CSV header depends on them).
- AuditLog EventType for ExportAppConfig confirmed (UNKNOWN — may need to add to Choice column).
- Admin utility flow rules understood — see `PACode/flows/admin-utility-flow-rules.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_ExportAppConfig`.
3. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varExportFileName | String | `concat('AppConfig_Export_', formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), '.csv')` |
| varCsvRows | String | `''` |
| varFileWebUrl | String | `''` |

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: UNKNOWN (use `SystemError` or add `ExportAppConfig` to EventType choices — confirm)
- eventCategory: `System`
- severity: `Info` / status: `Started`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `Pokretanje eksporta App Config.`

---

## Step 5 — Try scope

### 5a — Read all App Config items

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Top: 5000
- Order by: UNKNOWN (Category column ascending, then Key column ascending — if these columns exist)
- Select: UNKNOWN (select only the columns to include in the export — do not pull all internal SP columns)

Rename: `Get_AppConfigItems`

If the App Config list has more than 5000 items (unlikely), pagination is needed.
For v1, 5000 is sufficient — document this assumption.

### 5b — Build CSV header row

Add **Compose** — `Build_CsvHeader`:
```
concat(
  uriComponentToString('%EF%BB%BF'),
  'Category,Key,Value,Description,IsActive',
  decodeUriComponent('%0A')
)
```

This prepends the UTF-8 BOM and writes the header as the first line.
Column order is UNKNOWN — adjust to match actual App Config columns once internal names are confirmed.

Column names in the CSV header must match display names or agreed export labels — not internal column names.

### 5c — Apply to each App Config item

Add **Apply to each**: input = `body('Get_AppConfigItems')?['value']`

Add **Compose** — `Get_CurrentItem`:
```
item()
```

Build one CSV row per item. Each field must be CSV-escaped (see build notes for the escaping pattern).

Add **Compose** — `Build_CsvRow`:
```
concat(
  outputs('Escape_Category'), ',',
  outputs('Escape_Key'), ',',
  outputs('Escape_Value'), ',',
  outputs('Escape_Description'), ',',
  outputs('Escape_IsActive'),
  decodeUriComponent('%0A')
)
```

Where each `Escape_*` compose applies the CSV escape logic to the corresponding field value.
(See build notes for the full escape expression.)

Add **Compose** — `Append_CsvRow`:
```
concat(variables('varCsvRows'), outputs('Build_CsvRow'))
```

Set `varCsvRows` = `outputs('Append_CsvRow')`

**Performance note:** Concatenating a large string inside a loop is inefficient for large item counts. For App Config (expected < 500 items), this is acceptable. If performance is an issue, use Select action + join() — see build notes.

### 5d — Assemble final CSV content

Add **Compose** — `Build_CsvContent`:
```
concat(outputs('Build_CsvHeader'), variables('varCsvRows'))
```

---

## Step 6 — Save CSV to Exports library

Add **SharePoint — Create file**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Folder path: `/@parameters('gpdoccen_EV_DocCentralV3_docExports')`
  (UNKNOWN — confirm exact folder path format; may be `/sites/<site>/Exports` or just the library name)
- File name: `variables('varExportFileName')`
- File content: `outputs('Build_CsvContent')`

Rename: `Create_ExportFile`

The file content is passed as a string. Power Automate Create file accepts text content directly.
The UTF-8 BOM prepended in Step 5b ensures correct encoding.

Set `varFileWebUrl`:
```
body('Create_ExportFile')?['Path']
```
(UNKNOWN — the exact property name for the web URL in the Create file response. Test and confirm. See build notes.)

---

## Step 7 — Log Success

Call CF_DocCentralV3_LogEvent:
- severity: `Info` / status: `Success`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('App Config eksportovan u fajl: ', variables('varExportFileName'), '.')`

---

## Step 8 — Respond success

**Respond to a PowerApp or flow** with 5 outputs:

| Output | Value |
|---|---|
| success | `true` |
| message | `App Config je uspešno eksportovan.` |
| fileUrl | `variables('varFileWebUrl')` |
| fileName | `variables('varExportFileName')` |
| correlationId | `variables('varCorrelationId')` |
| errorCode | `''` |

---

## Step 9 — Catch scope

Configure Run after: failed, timed out.

Call CF_DocCentralV3_LogEvent:
- severity: `Error` / status: `Failed`
- errorCode: `EXPORT_FAILED`
- errorMessage: `result('Try')?[0]?['error']?['message']`
- correlationId: `variables('varCorrelationId')`

Respond failure:
- success: false
- message: `Greška pri eksportovanju App Config.`
- errorCode: `EXPORT_FAILED`
- fileUrl: `''`
- fileName: `''`
- correlationId: `variables('varCorrelationId')`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_ExportAppConfig`
- [ ] Trigger: Power Apps (V2) with 2 inputs
- [ ] 4 variables initialized
- [ ] Log Started (EventType confirmed)
- [ ] Get_AppConfigItems: Top 5000, ordered by Category/Key
- [ ] UTF-8 BOM prepended to CSV content
- [ ] CSV header matches actual App Config columns
- [ ] Apply to each builds one CSV row per item with proper escaping
- [ ] Empty App Config list: CSV created with header only (no crash)
- [ ] Create file: folder path correct for Exports library root
- [ ] varFileWebUrl set from Create file response
- [ ] Log Success before Respond
- [ ] Catch scope: log Error, respond failure
- [ ] All UNKNOWN column names resolved before building
- [ ] Flow inside DocCentralV3 solution
