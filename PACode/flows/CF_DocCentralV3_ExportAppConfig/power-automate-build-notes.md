# Power Automate build notes: CF_DocCentralV3_ExportAppConfig

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_ExportAppConfig |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog, EV_DocCentralV3_docExports |

---

## Why CSV, not Excel

Power Automate does not create new Excel (.xlsx) files from scratch.
The Excel connector can only write to an **existing** file (template copy approach).

CSV avoids this problem entirely:
- Generate a text string with comma-separated values.
- Save as `.csv` to SharePoint.
- Excel opens `.csv` files natively on double-click (Windows default behavior).
- No Excel connector, no OneDrive, no template files needed.

If Excel formatting is later required (headers bold, column widths, etc.), revisit with the template-copy approach (Option A in the design). For v1, CSV is the correct choice.

---

## UTF-8 BOM — required for Excel compatibility

Excel on Windows defaults to Windows-1252 encoding when opening CSV files.
Without the UTF-8 BOM, Serbian characters (č, ć, š, ž, đ) render as garbage.

Prepend the BOM using:
```
uriComponentToString('%EF%BB%BF')
```

This must be the very first character in the file content string — before any header text.

```
[Compose: Build_CsvHeader]
concat(uriComponentToString('%EF%BB%BF'), '"Category","Key",...', decodeUriComponent('%0A'))
```

Do NOT use `decodeBase64('77u/')` — that decodes to a string and may not produce the correct byte sequence when saved.
`uriComponentToString('%EF%BB%BF')` is the confirmed working approach.

---

## Building CSV rows: loop vs Select + join()

**Option A — Apply to each (simpler, used in implementation-steps.md):**
Concatenate rows in a loop using `varCsvRows` string variable.
Works for any item count but is slower for large sets due to string concatenation in a loop.

**Option B — Select + join() (better performance):**
Use the **Select** action to transform each App Config item to a CSV row string.
Then use `join(body('Select_CsvRows'), decodeUriComponent('%0A'))` to concatenate all rows at once.

```
[Select: Select_CsvRows]
From: body('Get_AppConfigItems')?['value']
Map:
  (no key — use the expression-only mode)
  Value: concat('"', replace(string(item()?['UNKNOWN_Category']), '"', '""'), '","',
                replace(string(item()?['UNKNOWN_Key']), '"', '""'), '","',
                replace(string(item()?['UNKNOWN_Value']), '"', '""'), '","',
                replace(string(item()?['UNKNOWN_Description']), '"', '""'), '","',
                string(item()?['UNKNOWN_IsActive']), '"')

[Compose: Build_CsvRows]
join(body('Select_CsvRows'), decodeUriComponent('%0A'))
```

Option B avoids the Apply to each loop entirely. Recommended for App Config exports where item count can be in the hundreds.

**Use Option B** — it runs faster and produces the same result without a loop.

---

## Select action — expression-only mode

The Select action has two modes:
- Key-value mode: map a key name to a value.
- Expression mode: select all values as a single expression (toggle with the expression switcher icon).

For CSV row building, use expression mode to produce the entire row as one string per item.
In expression mode, `item()` refers to the current array element.

---

## SharePoint Create file — folder path format

The folder path parameter for **SharePoint — Create file** expects the path relative to the site root:
```
/Exports
```
or for a document library with a different name:
```
/ExportsLibraryName
```

Do not include the site URL — just the relative path from the site root.

If `EV_DocCentralV3_docExports` stores the library name (e.g. `Exports`), the folder path is:
```
concat('/', parameters('gpdoccen_EV_DocCentralV3_docExports'))
```

Test with a known library name before finalizing. The Exports library root is the target — no subfolders.

---

## Create file response — file URL property

Test the actual property name after the first Create file action runs.
Common property paths:
- `body('Create_ExportFile')?['Path']` — relative server path (e.g. `/sites/DocCentral/Exports/file.csv`)
- `body('Create_ExportFile')?['{Link}']` — direct SharePoint URL (connector-specific)
- `body('Create_ExportFile')?['WebUrl']` — full URL (depends on connector version)

After the first test run, inspect the action output in the run history to find the correct property.

If the URL property is unreliable, return the `fileName` only and have Canvas App construct the URL:
```
concat(parameters('gpdoccen_EV_DocCentralV3_SharePointSite'), '/Exports/', variables('varExportFileName'))
```
This is acceptable since the Exports library path is known.

---

## Line ending — LF vs CRLF

CSV files technically use CRLF (`\r\n`) per RFC 4180.
Excel on Windows accepts both LF and CRLF.
Power Automate: use LF (`decodeUriComponent('%0A')`) — simpler.

If client reports that extra blank rows appear in Excel, switch to CRLF:
```
concat(decodeUriComponent('%0D'), decodeUriComponent('%0A'))
```

---

## Action naming convention

| Purpose | Name |
|---|---|
| Read App Config | `Get_AppConfigItems` |
| CSV header | `Build_CsvHeader` |
| Select rows (Option B) | `Select_CsvRows` |
| All rows joined | `Build_CsvRows` |
| Full CSV content | `Build_CsvContent` |
| Per-field escape | `Escape_Category`, `Escape_Key`, `Escape_Value`, etc. |
| Create file | `Create_ExportFile` |

---

## Checklist

- [ ] Get_AppConfigItems uses Top 5000
- [ ] UTF-8 BOM prepended as first character of file content
- [ ] CSV header labels match agreed column names
- [ ] CSV escaping applied to all text fields (commas, double-quotes, newlines)
- [ ] Select + join() used (Option B) rather than Apply to each loop
- [ ] Create file folder path correct for Exports library root
- [ ] varFileWebUrl set from Create file response (test property name after first run)
- [ ] Empty App Config result: creates header-only CSV, success returned
- [ ] AuditLog EventType confirmed (not UNKNOWN) before building
- [ ] Flow inside DocCentralV3 solution
