# Flow design: CF_DocCentralV3_ExportAppConfig

## Purpose

Exports all App Config items to an Excel file and saves it to the `Exports` SharePoint library.
Returns a download URL to the Canvas App so the administrator can download the file.

This is a utility/admin function used to back up or review the current codebook and
configuration state.

## Trigger type

Power Apps V2. Called directly from Canvas App (Administracija screen).

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_OneDrive` (or Excel connector — see design note)
- `gpdoccen_CR_DocCentralV3_Excel`

## Design note on Excel generation

Power Automate does not natively create a new Excel file from scratch via the Excel connector.
The recommended approach for generating an Excel export in Power Automate is one of:

**Option A — Template copy + populate**
A blank Excel template file exists in the Exports library.
The flow copies the template, then uses the Excel connector to add rows.

**Option B — CSV generation**
Generate a CSV string from App Config items and save as a `.csv` file in the Exports library.
CSV is readable in Excel without requiring the Excel connector for generation.
Simpler and more reliable than Excel generation.

**Recommended approach: Option B (CSV)**
Unless Excel formatting is required, CSV is preferred for reliability.
The file is named with a timestamp and saved in the root of the Exports library.

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`
- `gpdoccen_EV_DocCentralV3_docExports`

## Input schema

```json
{
  "initiatorEmail": "",
  "correlationId": ""
}
```

## Output schema

Success:
```json
{
  "success": true,
  "message": "App Config je uspešno eksportovan.",
  "fileUrl": "",
  "fileName": "",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Greška pri eksportovanju App Config.",
  "errorCode": "",
  "correlationId": ""
}
```

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varExportFileName` = `concat("AppConfig_Export_", formatDateTime(utcNow(), 'yyyyMMdd_HHmmss'), ".csv")`

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SystemError` — NOTE: no dedicated `ExportAppConfig` EventType exists in current AuditLog schema. Use closest available or `System`. UNKNOWN — confirm whether to add `ExportAppConfig` to EventType choices.
- EventCategory: `System`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje eksporta App Config."

**Step 3 — Read all App Config items**
Action: SharePoint — Get items
List: `EV_DocCentralV3_lstAppConfig`
Top: 5000 (or use paging if more than 5000 items — unlikely)
Order: by Category then Key (UNKNOWN column names)

Store results in variable `varAppConfigItems`.

If no items: return success with empty file (still create the file — it will just have headers).

**Step 4 — Build CSV content**

Compose CSV header row.
Column order for CSV (exact App Config column names UNKNOWN — use what is available):
```
Category,Key,Value,Description,IsActive
```
All column names are UNKNOWN until App Config schema is confirmed.

For each App Config item: compose one CSV row.
Escape commas and quotes in values (wrap fields containing commas or quotes in double-quotes,
escape internal double-quotes as `""`).

Concatenate all rows with newline `\n` separator.
Prepend UTF-8 BOM (`﻿`) so Excel opens the file correctly with special characters.

Store as `varCsvContent`.

**Step 5 — Save CSV file to Exports library**
Action: SharePoint — Create file
Site: `EV_DocCentralV3_SharePointSite`
Folder path: root of `EV_DocCentralV3_docExports`
File name: `varExportFileName`
File content: `varCsvContent` (as text, encoded as binary)

Store: `varFileWebUrl` = SharePoint file web URL from create file response.

**Step 6 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- Status: `Success`
- Message: `concat("App Config eksportovan u fajl: ", varExportFileName)`

**Step 7 — Return success response**
Return `fileUrl` = `varFileWebUrl` and `fileName` = `varExportFileName`.

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `EXPORT_FAILED`
- CorrelationId: `varCorrelationId`

Return failure response.

## File naming

Format: `AppConfig_Export_YYYYMMDD_HHmmss.csv`
Example: `AppConfig_Export_20260504_083015.csv`

Files are not cleaned up automatically. Old exports remain in the Exports library.
Cleanup is a manual admin task. If automated cleanup is needed, it is a future scope item.

## Security

Only admin users (as determined by `gblIsAdmin` in Canvas App) should see the Export button.
Canvas App enforces this via `Visible` property. Backend does not re-validate admin role —
this is acceptable as the export operation reads only App Config (non-sensitive config data).

If client requires backend admin check: read initiator's group membership from App Config
and verify against admin group (UNKNOWN key). Currently out of scope.

## Error codes

| Code | Meaning |
|---|---|
| EXPORT_FAILED | General failure during App Config read or file creation |
| APP_CONFIG_EMPTY | No App Config items found (returned as success with empty file) |

## Open items

| Item | Status |
|---|---|
| App Config column names for CSV header | UNKNOWN — depends on AppConfig.csv |
| AuditLog EventType for ExportAppConfig | UNKNOWN — not in current EventType list |
| Whether to add ExportAppConfig to EventType choices | TO BE CONFIRMED |
| Excel vs CSV approach — final decision | Recommended CSV; confirm with user |
| Export library cleanup policy | Not in scope for v1 |
| Backend admin group validation | Not in scope for v1 |

## Test scenarios

| Scenario | Expected result |
|---|---|
| Admin runs export | CSV file created in Exports library, URL returned |
| App Config has 200 items | All items in CSV, file created successfully |
| SharePoint file creation fails | Error returned, audit logged |
| App Config list is empty | Empty CSV with headers created, success returned |
| Special characters in values | Properly escaped in CSV output |
