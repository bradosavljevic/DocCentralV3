# Power Automate build notes: CF_DocCentralV3_SendReminders

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_SendReminders |
| Trigger | Recurrence (daily at 08:00 UTC) |
| Connection references | CR_DocCentralV3_SharePoint, CR_DocCentralV3_Outlook |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstPodsetnici, EV_DocCentralV3_lstAuditLog |

---

## Concurrency control — must configure manually after creation

Scheduled flows default to multiple concurrent runs allowed.
Immediately after creating this flow:

1. Open flow → Settings (gear icon at top right of flow editor).
2. Concurrency Control → toggle On.
3. Degree of parallelism = 1.
4. Save.

This prevents two simultaneous runs from double-sending reminders if the recurrence fires twice (reliability edge case).

---

## Recurrence trigger — no inputs

The Recurrence trigger has no body. Do not reference `triggerBody()` — it is always null.
All per-run state starts from initialized variables.

To get today's date:
```
formatDateTime(utcNow(), 'yyyy-MM-dd')
```

For timezone-adjusted date (Central European Time example):
```
convertTimeZone(utcNow(), 'UTC', 'Central Europe Standard Time', 'yyyy-MM-dd')
```

The timezone string must match exactly a Windows timezone name.
Common names: `'Central Europe Standard Time'`, `'W. Europe Standard Time'`, `'Serbia Standard Time'` (UNKNOWN — test in expression editor).

---

## OData filter for Yes/No (Boolean) columns

SharePoint Yes/No columns use integer values in OData filters:
- `eq 0` = false (not sent)
- `eq 1` = true (sent)

Example filter:
```
UNKNOWN_ReminderDate eq '2026-05-04' and UNKNOWN_IsSent eq 0
```

Do not use `eq false` or `eq 'false'` — these will not work in SharePoint OData.

---

## RecipientEmails — Text field parse

If RecipientEmails is stored as a semicolon-separated Text column:

```
[Compose: Parse_RecipientList]
split(item()?['UNKNOWN_RecipientEmails'], ';')
```

This returns an array of strings. Each entry may have leading/trailing whitespace.

Filter empty entries and trim:
```
[Compose: Filter_ValidRecipients]
filter(outputs('Parse_RecipientList'), not(empty(trim(item()))))
```

Reference email in inner loop: `trim(item())`

Note: Power Automate `split()` returns an array. If the source value is null or empty, `split(null, ';')` returns `['']` — one empty string. The `filter(not(empty(trim(item()))))` handles this correctly.

---

## RecipientEmails — People field parse

If RecipientEmails is a People (multi-value) field, SharePoint returns it as an array of person objects in the Get items response. Each object has:
```json
{ "DisplayName": "...", "Email": "user@org.rs", "JobTitle": "..." }
```

Extract emails:
```
[Compose: Extract_Emails]
map(item()?['UNKNOWN_RecipientEmails'], item()?['Email'])
```

Note: In a People field reference inside Apply to each, the outer `item()` is the reminder item. The inner `item()` in the `map()` expression refers to each person object. This nested `item()` reference can cause confusion — use a Compose to store the People array first:

```
[Compose: Get_PeopleArray]
item()?['UNKNOWN_RecipientEmails']

[Compose: Extract_Emails]
map(outputs('Get_PeopleArray'), item()?['Email'])
```

---

## Inner Apply to each — email send per recipient

The inner loop iterates over `outputs('Filter_ValidRecipients')` — an array of email address strings.

Inside the inner loop, `item()` = the current email string.

Configure the Outlook Send email action's **Run after**: succeeded only (let failures bubble to the outer condition check).
Then after the action, add a condition: `equals(actions('Send_ReminderEmail')?['status'], 'Failed')`.

Do not configure the Outlook action itself to Run after failed — that would cause the action to run on its own failure, which is circular. Check the failure with a subsequent Condition instead.

---

## Marking IsSent — always run

The **Update item** (IsSent = true) action must always run, even if email sending failed.
This prevents infinite reprocessing of a reminder with broken recipients.

Configure the Update item action **Run after**: succeeded and failed (check both checkboxes in the Run after dialog).

If Update item itself fails, log a Warning only — do not terminate. The reminder will be reprocessed next day (acceptable trade-off). This is documented in the design as acceptable behavior.

---

## No Terminate Succeeded in scheduled flows

Scheduled flows do not have Respond actions. There is no need to use Terminate Succeeded to avoid double-Respond.

To exit the loop early on per-item failures, use nested conditions and the Scope pattern (configure subsequent actions in the scope with Run after including Skipped).

For the overall "no reminders" case, simply do not enter the Apply to each — use a Condition on the length of the Get_DueReminders result.

---

## Apply to each — Do not use concurrency on the outer loop

The outer Apply to each processes reminders sequentially (default concurrency = 1).
Do not enable concurrency on the outer Apply to each — sequential processing is correct here:
- varSentCount and varFailedCount are incremented in the loop. Concurrent execution causes race conditions on variable increments.
- SharePoint Update item for IsSent is safe sequentially but not required to be concurrent.

---

## Outlook Send an email (V2) — correct action

Use **Office 365 Outlook — Send an email (V2)** via the CR_DocCentralV3_Outlook connection reference.
Do not use the older "Send an email" action — it uses a different connector version.

The `To` field accepts a single email address string or a semicolon-separated list.
For per-recipient loops, pass a single address per iteration.

---

## Action naming convention

| Purpose | Name |
|---|---|
| Read due reminders | `Get_DueReminders` |
| Current reminder reference | `Get_CurrentReminder` |
| Parse recipient list | `Parse_RecipientList` |
| Filter valid recipients | `Filter_ValidRecipients` |
| Send email | `Send_ReminderEmail` |
| Mark as sent | `Update_Reminder_MarkSent` |
| Per-reminder scope | `Scope_ProcessReminder` |

---

## Checklist

- [ ] Recurrence trigger: daily, 08:00 UTC
- [ ] Concurrency Control = 1 in flow Settings (not the trigger config — in Settings tab)
- [ ] varTodayDate uses correct timezone expression
- [ ] OData filter uses `eq 0` for IsSent = false (not `eq false`)
- [ ] Top: 500 on Get_DueReminders
- [ ] RecipientEmails parsing matches actual column type (Text or People)
- [ ] Inner loop: email Send action run after: succeeded only
- [ ] Inner loop: failure checked with Condition after Send action
- [ ] Update_Reminder_MarkSent runs after succeeded AND failed
- [ ] Update_Reminder_MarkSent failure is Warning only
- [ ] varSentCount and varFailedCount at reminder level, not recipient level
- [ ] No Terminate actions needed in this flow
- [ ] Flow inside DocCentralV3 solution
