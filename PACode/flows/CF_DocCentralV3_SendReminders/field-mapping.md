# Field mapping: CF_DocCentralV3_SendReminders

## Podsetnici — fields read

All internal column names UNKNOWN until list schema confirmed.

| Display name | Internal name | Expression | Notes |
|---|---|---|---|
| (ID) | ID | `item()?['ID']` | Standard SP column |
| Title | Title | `item()?['Title']` | Standard SP column |
| ReminderDate | UNKNOWN | `item()?['UNKNOWN_ReminderDate']` | Filter: eq varTodayDate |
| RecipientEmails | UNKNOWN | `item()?['UNKNOWN_RecipientEmails']` | See parsing section below |
| CreatedByEmail | UNKNOWN | `item()?['UNKNOWN_CreatedByEmail']` | Written to AuditLog |
| IsSent | UNKNOWN | `item()?['UNKNOWN_IsSent']` | Filter: eq false/0 |

---

## Podsetnici — fields written

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| IsSent | UNKNOWN | `true` | Set after email sent (or attempted) |
| SentAt | UNKNOWN (proposed) | `utcNow()` | Optional — set alongside IsSent |

Written via **SharePoint — Update item** (standard connector, not HTTP PATCH — no ETag needed here).

---

## OData filter for due reminders

```
UNKNOWN_ReminderDate eq '<varTodayDate>' and UNKNOWN_IsSent eq 0
```

Where:
- `<varTodayDate>` = `formatDateTime(utcNow(), 'yyyy-MM-dd')` (or timezone-converted)
- `0` = false for Yes/No field in OData filter syntax
- Replace `UNKNOWN_ReminderDate` and `UNKNOWN_IsSent` with actual internal column names

If the ReminderDate column stores date+time (not date-only), the filter must account for time:
```
UNKNOWN_ReminderDate ge '<varTodayDate>T00:00:00Z' and UNKNOWN_ReminderDate lt '<varTomorrowDate>T00:00:00Z'
```
Where `varTomorrowDate` = `formatDateTime(addDays(utcNow(), 1), 'yyyy-MM-dd')`

---

## RecipientEmails field — parsing by type

### Option A: Text field (semicolon-separated)

```
[Compose: Parse_RecipientList]
split(item()?['UNKNOWN_RecipientEmails'], ';')

[Compose: Filter_ValidRecipients]
filter(outputs('Parse_RecipientList'), not(empty(trim(item()))))
```

Use `trim(item())` when referencing each email address in the inner loop.

### Option B: People field (multi-value)

SharePoint People field returns an array of objects. Each object has an `Email` property.

```
[Compose: Parse_RecipientList]
item()?['UNKNOWN_RecipientEmails']

[Compose: Extract_Emails]
map(outputs('Parse_RecipientList'), item()?['Email'])

[Compose: Filter_ValidRecipients]
filter(outputs('Extract_Emails'), not(empty(item())))
```

Confirm field type and update expressions accordingly before building.

---

## Email template

Subject: `concat('DocCentral podsetnik: ', item()?['Title'])`

Body (HTML):
```html
<p>Poštovani,</p>
<p>Imate podsetnik u sistemu DocCentral.</p>
<p><strong>Opis:</strong> {Title}</p>
<p><strong>Datum podsetnika:</strong> {ReminderDate}</p>
<p><strong>Kreirao:</strong> {CreatedByEmail}</p>
<p>Ovu poruku je generisao sistem DocCentral automatski.</p>
```

Body template is UNKNOWN — confirm with client for final format and branding.
The placeholders above are functional defaults.

---

## Open items

| Item | Status |
|---|---|
| Podsetnici ReminderDate internal column name | UNKNOWN |
| Podsetnici ReminderDate column type (Date vs DateTime) | UNKNOWN — affects OData filter |
| Podsetnici RecipientEmails internal column name | UNKNOWN |
| Podsetnici RecipientEmails field type (Text vs People) | UNKNOWN — affects parse logic |
| Podsetnici CreatedByEmail internal column name | UNKNOWN |
| Podsetnici IsSent internal column name | UNKNOWN — may need to be created |
| Podsetnici SentAt internal column name | UNKNOWN — proposed field; may not exist yet |
| Reminder timezone App Config key | UNKNOWN |
| Email body template | UNKNOWN — use default until client confirms |
| Maximum reminders per day | UNKNOWN — assumed < 500; increase Top if needed |
