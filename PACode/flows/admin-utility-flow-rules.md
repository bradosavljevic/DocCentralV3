# Admin and utility flow rules: shared reference

## Flow inventory

| Flow | Trigger | Purpose |
|---|---|---|
| CF_DocCentralV3_SendReminders | Scheduled (daily) | Email due reminders from Podsetnici list |
| CF_DocCentralV3_ExportAppConfig | Power Apps V2 | Export App Config to CSV file in Exports library |
| CF_DocCentralV3_GenerateArchiveBookPdf | Power Apps V2 | Generate HTML archive book for a registry year |

---

## Podsetnici list — required fields

All internal column names UNKNOWN until schema confirmed.

| Display name | Type | Internal name | Purpose | Notes |
|---|---|---|---|---|
| Title | Text | Title (default) | Reminder description | Standard SP column |
| ReminderDate | Date | UNKNOWN | Date the reminder fires | Filter: eq today, IsSent = false |
| RecipientEmails | Text or People | UNKNOWN | Email addresses of recipients | If Text: semicolon-separated. If People: JSON array |
| CreatedByEmail | Text | UNKNOWN | Who created the reminder | Written to AuditLog |
| IsSent | Yes/No | UNKNOWN | Whether email was sent | Prevents resend on rerun |
| SentAt | Date/Time | UNKNOWN (proposed) | When it was sent | Optional — set with IsSent |

If `IsSent` and `SentAt` columns do not exist in the list, they must be created before building SendReminders.
These are required for idempotency.

### RecipientEmails field type decision

- **If Text (multi-line)**: Parse with `split(field, ';')`. Trim each element. Filter empty strings.
- **If People (multi-value)**: Parse JSON. Extract `email` property from each entry. Apply `toLower()`.

Confirm field type from list schema before building. Both paths are documented in SendReminders build notes.

---

## Exports library

All three utility flows write to or read from the Exports library.

| Flow | File type | Naming pattern |
|---|---|---|
| ExportAppConfig | .csv | `AppConfig_Export_{YYYYMMDD_HHmmss}.csv` |
| GenerateArchiveBookPdf | .html | `ArhivskaKnjiga_{year}_{YYYYMMDD_HHmmss}.html` |

Folder path: root of `EV_DocCentralV3_docExports`. No subfolders.
No automatic cleanup — admin responsibility.
EV: `@parameters('gpdoccen_EV_DocCentralV3_docExports')`

### Getting the file web URL after Create file

The SharePoint Create file action returns the file metadata. The web URL is at:
```
body('Create_File')?['Path']
```
or assembled as:
```
concat(
  parameters('gpdoccen_EV_DocCentralV3_SharePointSite'),
  '/sites/...',  — UNKNOWN exact path
  body('Create_File')?['Name']
)
```
The exact property depends on the connector version. Test after first file creation to confirm.
Alternatively, return the file name and let the Canvas App construct the URL using the known Exports library path.

---

## CSV generation — field escaping rules

All values written to CSV must be escaped:

| Condition | Action |
|---|---|
| Value contains comma | Wrap entire value in double quotes: `"value,with,commas"` |
| Value contains double-quote | Escape as `""` inside quoted value: `"value with ""quotes""` |
| Value contains newline | Replace `\n` and `\r` with space before writing |
| Value is null or empty | Write empty field (two consecutive commas or trailing comma) |

In Power Automate expressions, a reusable escape function does not exist natively.
Use nested `replace()` calls:
```
replace(replace(replace(item()?['UNKNOWN_Value'], '"', '""'), decodeUriComponent('%0A'), ' '), decodeUriComponent('%0D'), ' ')
```
Then wrap in quotes if the value contains a comma.

UTF-8 BOM prefix for CSV: `uriComponentToString('%EF%BB%BF')`

---

## HTML generation — encoding rules

For GenerateArchiveBookPdf, all values from SharePoint must be HTML-escaped before insertion:

| Character | Escaped form |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |

Power Automate expression for basic HTML escaping (no built-in function):
```
replace(replace(replace(replace(value, '&', '&amp;'), '<', '&lt;'), '>', '&gt;'), '"', '&quot;')
```

Apply in a **Compose** action per field, or in the row compose expression directly.

UTF-8 BOM prefix for HTML: `uriComponentToString('%EF%BB%BF')`

---

## Date formatting

| Purpose | Expression |
|---|---|
| Today (for reminder filter) | `formatDateTime(utcNow(), 'yyyy-MM-dd')` |
| Display date (Serbian) | `formatDateTime(value, 'dd.MM.yyyy')` |
| File timestamp suffix | `formatDateTime(utcNow(), 'yyyyMMdd_HHmmss')` |

When converting timezone for the reminder date filter:
```
convertTimeZone(utcNow(), 'UTC', 'Central Europe Standard Time', 'yyyy-MM-dd')
```
Timezone name is UNKNOWN — confirm from App Config or client. Default to UTC if not configured.

---

## Scheduled trigger — concurrency and idempotency

SendReminders runs on a Recurrence trigger. Power Automate cloud flows can fire more than once
for a single scheduled interval in rare reliability scenarios.

**Defense**: filter Podsetnici by `IsSent = false`. Mark as sent immediately after email dispatch.
This ensures each reminder is sent at most once even on double-fire.

**Concurrency setting**: Set the scheduled flow to run with concurrency = 1 (singleton).
Power Automate: go to flow settings → Concurrency Control → On, Degree of parallelism = 1.
This prevents two simultaneous runs from processing the same reminder.

---

## AuditLog EventType coverage

| Flow | EventType | Notes |
|---|---|---|
| SendReminders | `SendReminder` | Confirm this EventType exists in AuditLog choices |
| ExportAppConfig | UNKNOWN | No dedicated EventType in current schema — confirm whether to add `ExportAppConfig` |
| GenerateArchiveBookPdf | `GenerateArchiveBookPdf` | Confirm this EventType exists in AuditLog choices |

If AuditLog EventType is a Choice column, new values must be added to the column before the flow can log them.

---

## Open items (shared)

| Item | Status |
|---|---|
| Podsetnici ReminderDate internal column name | UNKNOWN |
| Podsetnici RecipientEmails column type and internal name | UNKNOWN |
| Podsetnici IsSent internal column name | UNKNOWN — may not exist yet |
| Podsetnici SentAt internal column name | UNKNOWN — proposed field |
| Podsetnici CreatedByEmail internal column name | UNKNOWN |
| Reminder timezone App Config key | UNKNOWN |
| Reminder CRUD access model (Canvas App Patch vs flow) | TO BE DECIDED |
| Exports library root folder path for Create file | UNKNOWN — confirm folder path format |
| Create file action property for web URL | UNKNOWN — test after first creation |
| App Config column names for CSV export | UNKNOWN — all UNKNOWN |
| App Config EventType for ExportAppConfig in AuditLog | UNKNOWN — not in current list |
| Svi predmeti internal column names for archive book | UNKNOWN — all UNKNOWN |
| Year filter strategy for archive book (column vs date range) | UNKNOWN |
| Sort column for DelovodniBroj | UNKNOWN |
| Organization name App Config key | UNKNOWN |
