# Field mapping: CF_DocCentralV3_ExportAppConfig

## App Config — fields read

All internal column names UNKNOWN. The CSV export will include a subset of columns.
The exact column list must be confirmed from the App Config list schema.

| Display name (assumed) | Internal name | CSV header label | Notes |
|---|---|---|---|
| Category | UNKNOWN | `Category` | Grouping/type of config entry |
| Key | UNKNOWN | `Key` | Unique identifier within category |
| Value | UNKNOWN | `Value` | Config value — may contain commas; must be escaped |
| Description | UNKNOWN | `Description` | Human-readable description — may be absent |
| IsActive | UNKNOWN | `IsActive` | Active flag — may be Yes/No or text |

Additional columns may exist. Confirm full schema from App Config list before writing CSV header.

OData `$select` expression (after column names confirmed):
```
$select=UNKNOWN_Category,UNKNOWN_Key,UNKNOWN_Value,UNKNOWN_Description,UNKNOWN_IsActive
```

Do not select all columns (`$select=*`) — this includes internal SP metadata columns that are not meaningful in a CSV export.

---

## App Config — Get items expression

```
[Get_AppConfigItems]
Site: @parameters('gpdoccen_EV_DocCentralV3_SharePointSite')
List: @parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')
Top: 5000
Filter: (none — export all items)
Order by: UNKNOWN_Category asc, UNKNOWN_Key asc (after column names confirmed)
Select: (list of columns above — after internal names confirmed)
```

---

## CSV escaping expressions per field

Each field requires a Compose action to apply CSV escaping before being included in the row.
The escape sequence wraps a field in quotes if it contains commas, double-quotes, or newlines.

Full CSV-safe escape expression for a single field value:

```
[Compose: Escape_Value]
if(
  or(
    contains(string(item()?['UNKNOWN_Value']), ','),
    contains(string(item()?['UNKNOWN_Value']), '"'),
    contains(string(item()?['UNKNOWN_Value']), decodeUriComponent('%0A'))
  ),
  concat('"', replace(string(item()?['UNKNOWN_Value']), '"', '""'), '"'),
  if(empty(item()?['UNKNOWN_Value']), '', string(item()?['UNKNOWN_Value']))
)
```

Apply the same pattern to each column. Use null-safe `string(item()?['UNKNOWN_*'])` — `string()` converts null to an empty string.

**Simplified version (always quote all fields):**
For simplicity, all fields can be unconditionally wrapped in double-quotes with internal quotes escaped:
```
concat('"', replace(if(empty(item()?['UNKNOWN_Value']), '', string(item()?['UNKNOWN_Value'])), '"', '""'), '"')
```
This is slightly less readable in Excel but always safe. Acceptable for an admin export.

---

## CSV row format

```
[Compose: Build_CsvRow]
concat(
  outputs('Escape_Category'), ',',
  outputs('Escape_Key'), ',',
  outputs('Escape_Value'), ',',
  outputs('Escape_Description'), ',',
  outputs('Escape_IsActive'),
  decodeUriComponent('%0A')
)
```

Line ending: `decodeUriComponent('%0A')` = LF (`\n`).
Excel on Windows expects CRLF, but opens LF-terminated files correctly.
Use LF unless client reports issues.

---

## CSV header

```
[Compose: Build_CsvHeader]
concat(
  uriComponentToString('%EF%BB%BF'),
  '"Category","Key","Value","Description","IsActive"',
  decodeUriComponent('%0A')
)
```

The UTF-8 BOM (`uriComponentToString('%EF%BB%BF')`) must be the very first byte of the file.
It is prepended here on the header line (the first line output to the file).

---

## Exports library — Create file

| Property | Value |
|---|---|
| Site | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| Folder path | UNKNOWN — root of `@parameters('gpdoccen_EV_DocCentralV3_docExports')`. Confirm exact path format (e.g. `/Exports` or the full relative path). |
| File name | `variables('varExportFileName')` = `AppConfig_Export_YYYYMMDD_HHmmss.csv` |
| File content | `outputs('Build_CsvContent')` |

---

## File web URL — from Create file response

The exact property name is UNKNOWN — test after first file creation:

```
body('Create_ExportFile')?['Path']
```

or, if Path is a relative path, construct the full URL:
```
concat(
  parameters('gpdoccen_EV_DocCentralV3_SharePointSite'),
  body('Create_ExportFile')?['Path']
)
```

Alternative: return only the `fileName` in the response and let the Canvas App construct the URL using the known Exports library base path.

---

## Open items

| Item | Status |
|---|---|
| App Config internal column names for all exported columns | UNKNOWN |
| App Config column list — which columns to include in export | UNKNOWN |
| AuditLog EventType for ExportAppConfig | UNKNOWN — confirm or add to Choice column |
| Exports library folder path format for Create file | UNKNOWN |
| Create file response property for web URL | UNKNOWN — test after first creation |
| Whether to use Select on Get items (performance) | Recommended — list columns once confirmed |
