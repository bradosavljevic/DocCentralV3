# Flow design: CF_DocCentralV3_SendReminders

## Purpose

Scheduled flow that runs daily and sends reminder emails to recipients for reminders
whose date matches today. Each reminder is sent exactly once — on the configured date.

This flow does not process individual reminder CRUD operations (create/edit/delete).
It only handles the daily dispatch of due reminders.

## Trigger type

Scheduled (Recurrence). Runs daily at a configured time.

Default send time: **08:00 UTC** (or per App Config — UNKNOWN key for reminder send time offset).

The exact UTC time depends on the client's timezone. This should be read from App Config
if timezone configuration exists (UNKNOWN). If not configured, default to 08:00 UTC.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`
- `gpdoccen_CR_DocCentralV3_Outlook`
- `gpdoccen_CR_DocCentralV3_Office365Users`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstPodsetnici`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Podsetnici list schema (PARTIAL — internal column names UNKNOWN)

Documented rules from `data-model/podsetnici.md`:
- Reminder sends email once on the date defined in the reminder item.
- Default send time is 08:00.
- One reminder can target one or multiple recipients.
- User can create, edit, delete reminders.

Required columns (internal names UNKNOWN):

| Display name | Type | Notes |
|---|---|---|
| Title | Text | Description of the reminder |
| ReminderDate | Date | The date the reminder should be sent |
| RecipientEmails | Text (multi-line) or People | Semicolon or comma-separated emails, or People field |
| CreatedByEmail | Text | Who created the reminder |
| IsSent | Yes/No | Whether the email has been sent |

`IsSent` flag is used to prevent resending on reruns or if the scheduled flow fires more than once on the same day. UNKNOWN column name.

## Input

No input parameters (scheduled trigger). Correlation ID is generated per run.

## Output

No output (scheduled flow). Results are written to AuditLog only.

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = `guid()`
- `varTodayDate` = `convertTimeZone(utcNow(), 'UTC', 'Central Europe Standard Time', 'yyyy-MM-dd')` — UNKNOWN: exact timezone depends on client App Config. Use UTC date if timezone not configured.
- `varSentCount` (Integer) = 0
- `varFailedCount` (Integer) = 0

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendReminder`
- EventCategory: `Reminder`
- Severity: `Info`
- Status: `Started`
- CorrelationId: `varCorrelationId`
- Message: `concat("Pokretanje slanja podsetnika za datum ", varTodayDate)`

**Step 3 — Read due reminders from Podsetnici**
Action: SharePoint — Get items
List: `EV_DocCentralV3_lstPodsetnici`
Filter: `ReminderDate eq '<varTodayDate>'` AND `IsSent eq 0` (or `false`)

Top: use pagination if more than 5000 items possible (unlikely but apply defensive paging).

If no items: log Info (no reminders due). Return without error.

**Step 4 — Apply to each reminder item**

For each item, within a per-item error scope:

**Step 4a — Parse recipient list**
Read `RecipientEmails` column (UNKNOWN internal name).
Parse into individual email addresses:
- If stored as semicolon-separated string: `split(item.RecipientEmails, ';')`
- If stored as People field (JSON array): extract `Email` property from each entry.
- Trim whitespace from each address.
- Filter out empty strings.

If recipient list is empty:
- Log warning (reminder has no recipients).
- Mark as sent to prevent reprocessing (see Step 4c).
- Continue to next item.

**Step 4b — Send email to each recipient**
For each recipient email in the parsed list:

Action: Outlook — Send an email (V2)
- To: recipient email
- Subject: `concat("DocCentral podsetnik: ", item.Title)`
- Body: Reminder details including `Title`, `ReminderDate`, and who created the reminder.
  Email body template UNKNOWN — use a simple informational format.
- From / Reply-To: not configured (uses the Outlook connection's mailbox — typically service account).

If email send fails for one recipient: log warning per recipient. Continue sending to others.
Track failure per reminder item.

**Step 4c — Mark reminder as sent**
Action: SharePoint — Update item
List: `EV_DocCentralV3_lstPodsetnici`
ID: item ID
Set `IsSent` = true (UNKNOWN internal column name)

This update prevents resending if the flow reruns on the same day.

If update fails: log warning. The reminder may be resent on next run — acceptable trade-off
vs failing the entire batch.

**Step 4d — Log per-reminder event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendReminder`
- Severity: `Info`
- Status: `Success`
- UserEmail: item `CreatedByEmail` (UNKNOWN internal name)
- CorrelationId: `varCorrelationId`
- Message: `concat("Podsetnik '", item.Title, "' je poslat.")`

Increment `varSentCount`.

On per-item error: increment `varFailedCount`. Log Error per item. Continue with next item.

### Finalization

**Step 5 — Log overall result**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendReminder`
- Severity: `Info` if no failures, `Warning` if some failed
- Status: `Success` if no failures, `Failed` if any failed
- CorrelationId: `varCorrelationId`
- Message: `concat("Slanje podsetnika završeno. Poslato: ", varSentCount, ", Neuspešno: ", varFailedCount)`

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `SendReminder`
- Severity: `Error`
- Status: `Failed`
- CorrelationId: `varCorrelationId`
- Message: "Neočekivana greška pri slanju podsetnika."

## Idempotency

The `IsSent` flag ensures that even if the scheduled trigger fires twice on the same day
(Power Automate reliability behavior), reminders are not sent twice.
Reminders are filtered by `IsSent = false` before processing.

## Reminder CRUD (not in this flow)

Creating, editing, and deleting reminders is handled in the Canvas App.

Options for write access to `Podsetnici` list:
- **Option A**: Users have Write access to `Podsetnici` list — Canvas App uses `Patch()` directly.
- **Option B**: Write access restricted to service account — Canvas App calls a dedicated flow.

**Decision needed**: UNKNOWN — to be confirmed. If users are given Write access to `Podsetnici`
only (as an exception to the general Read Only rule), Option A is simpler and acceptable.
If security policy requires all writes via service account, a separate reminder CRUD flow is needed
(not in current inventory — to be added if required).

## Open items

| Item | Status |
|---|---|
| Podsetnici RecipientEmails column — type and internal name | UNKNOWN |
| Podsetnici IsSent column — internal name | UNKNOWN |
| Podsetnici ReminderDate column — internal name | UNKNOWN |
| Podsetnici CreatedByEmail column — internal name | UNKNOWN |
| App Config key for reminder send timezone/time offset | UNKNOWN |
| Reminder CRUD access model (direct Patch vs flow) | TO BE DECIDED |
| Email body template | UNKNOWN |
| Maximum reminders per day (pagination threshold) | UNKNOWN — assume < 500 for now |

## Test scenarios

| Scenario | Expected result |
|---|---|
| One reminder due today, one recipient | Email sent, IsSent = true, audit logged |
| One reminder, multiple recipients | All receive email |
| No reminders due today | No emails, logged as Info |
| Reminder has no recipients | Warning logged, marked as sent |
| Email delivery fails | Warning per recipient, IsSent still updated |
| Flow runs twice same day | Second run finds IsSent = true, skips |
| IsSent update fails | Warning logged, possible resend on next run |
