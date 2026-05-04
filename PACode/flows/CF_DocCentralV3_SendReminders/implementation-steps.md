# Implementation steps: CF_DocCentralV3_SendReminders

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- Podsetnici list exists with columns: ReminderDate, RecipientEmails, CreatedByEmail, IsSent.
  If `IsSent` or `SentAt` columns do not exist, add them before building this flow.
- Confirm RecipientEmails column type (Text/multi-line vs People) — parsing logic differs.
- Confirm internal column names for all Podsetnici fields (all UNKNOWN).
- AuditLog EventType `SendReminder` exists (add to Choice column if missing).
- Connection references CR_DocCentralV3_SharePoint and CR_DocCentralV3_Outlook ready.
- Utility flow rules understood — see `PACode/flows/admin-utility-flow-rules.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Scheduled.
2. Name: `CF_DocCentralV3_SendReminders`.
3. Trigger: Recurrence.
4. After creation: go to flow Settings → Concurrency Control → On → Degree of parallelism = 1.
   This prevents two simultaneous runs from double-sending reminders.

---

## Step 2 — Trigger: Recurrence

Configure:
- Frequency: Day
- Interval: 1
- Start time: Set to today's date at `08:00:00` UTC (or per App Config timezone offset — UNKNOWN).
- Time zone: UTC (the flow runs at a fixed UTC time; timezone conversion happens inside the flow).

The trigger has no inputs and no outputs used by the flow body.

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `guid()` |
| varTodayDate | String | `formatDateTime(utcNow(), 'yyyy-MM-dd')` |
| varSentCount | Integer | `0` |
| varFailedCount | Integer | `0` |

**Timezone note for varTodayDate:**
If the client's timezone is ahead of UTC (e.g. Central European Time = UTC+1 or UTC+2 in summer),
the reminder date in SharePoint may be stored as a local date. Use:
```
convertTimeZone(utcNow(), 'UTC', 'Central Europe Standard Time', 'yyyy-MM-dd')
```
UNKNOWN — confirm timezone from App Config or client before finalizing.
Default to `formatDateTime(utcNow(), 'yyyy-MM-dd')` (UTC) until confirmed.

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `SendReminder` / eventCategory: `Reminder`
- severity: `Info` / status: `Started`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Pokretanje slanja podsetnika za datum ', variables('varTodayDate'), '.')`

---

## Step 5 — Try scope

### 5a — Read due reminders from Podsetnici

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstPodsetnici')`
- Filter: `UNKNOWN_ReminderDate eq '<varTodayDate>' and UNKNOWN_IsSent eq 0`
  - (UNKNOWN: internal column names for both fields)
  - Date format in OData filter: `'yyyy-MM-dd'` — confirm SharePoint stores date as ISO string
  - IsSent is Yes/No: use `eq 0` for false, `eq 1` for true in OData filter
- Top: 500 (reminder count per day is expected to be low; revisit if needed)

Rename: `Get_DueReminders`

Add **Condition**: `equals(length(body('Get_DueReminders')?['value']), 0)`

Yes (no reminders due):
1. Call LogEvent — Info: `concat('Nema podsetnika za slanje za datum ', variables('varTodayDate'), '.')`
2. **Do not terminate — proceed to finalization.**

### 5b — Apply to each reminder

(Only runs if there are due reminders — place inside the No branch of the condition above, or use a condition on the length before the Apply to each.)

Add **Apply to each**: input = `body('Get_DueReminders')?['value']`

Inside the loop, add a **Scope** named `Scope_ProcessReminder` configured to Run after succeeded and failed. This allows per-reminder error accumulation without stopping the loop.

#### 5b-i — Store current item reference

Add **Compose** — `Get_CurrentReminder`:
```
item()
```

#### 5b-ii — Parse recipient emails

Add **Compose** — `Get_RawRecipients`:
```
outputs('Get_CurrentReminder')?['UNKNOWN_RecipientEmails']
```

**If RecipientEmails is Text (semicolon-separated):**

Add **Compose** — `Parse_RecipientList`:
```
split(outputs('Get_RawRecipients'), ';')
```

Add **Compose** — `Filter_ValidRecipients`:
```
filter(outputs('Parse_RecipientList'), not(empty(trim(item()))))
```

**If RecipientEmails is People field (JSON array from SharePoint):**

Add **Compose** — `Parse_RecipientList`:
```
body('Get_DueReminders')?['value']
```
Extract emails: `map(outputs('Get_RawRecipients'), item()?['Email'])`

Confirm field type before building. Document which path was used in build notes.

Add **Condition**: `equals(length(outputs('Filter_ValidRecipients')), 0)`

Yes (no valid recipients):
1. Call LogEvent — Warning: `concat('Podsetnik ID ', string(outputs('Get_CurrentReminder')?['ID']), ' nema validne primaoce.')`
2. Continue to Step 5b-iv (mark as sent to prevent reprocessing).

#### 5b-iii — Send email to each recipient

Add **Apply to each** (inner loop): input = `outputs('Filter_ValidRecipients')`

Inside inner loop:

Add **Outlook — Send an email (V2)**:
- To: `trim(item())` (for Text field) or `item()?['Email']` (for People field)
- Subject: `concat('DocCentral podsetnik: ', outputs('Get_CurrentReminder')?['Title'])`
- Body (HTML):
```html
<p>Poštovani,</p>
<p>Imate podsetnik u sistemu DocCentral.</p>
<p><strong>Opis:</strong> @{outputs('Get_CurrentReminder')?['Title']}</p>
<p><strong>Datum podsetnika:</strong> @{outputs('Get_CurrentReminder')?['UNKNOWN_ReminderDate']}</p>
<p><strong>Kreirao:</strong> @{outputs('Get_CurrentReminder')?['UNKNOWN_CreatedByEmail']}</p>
<p>Ovu poruku je generisao sistem DocCentral automatski.</p>
```

Rename: `Send_ReminderEmail`

Configure **Run after** on the Send action: succeeded and failed.
After send, check if action failed:

Add **Condition**: `equals(actions('Send_ReminderEmail')?['status'], 'Failed')`

Yes (email failed):
1. Call LogEvent — Warning: `concat('Slanje podsetnika ID ', string(outputs('Get_CurrentReminder')?['ID']), ' nije uspelo za primaoca: ', item(), '.')`
2. Set `varFailedCount` = `add(variables('varFailedCount'), 1)` (increment once per failed reminder, not per recipient).

Note: increment varFailedCount on the **reminder item** level after the inner loop, not per recipient.

#### 5b-iv — Mark reminder as sent

Add **SharePoint — Update item**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstPodsetnici')`
- ID: `outputs('Get_CurrentReminder')?['ID']`
- Fields (UNKNOWN internal column names):
  - `UNKNOWN_IsSent` = `true`

Rename: `Update_Reminder_MarkSent`

Configure **Run after**: succeeded and failed (always attempt to mark sent even if email failed).

If update fails:
1. Call LogEvent — Warning: `concat('Označavanje podsetnika ID ', string(outputs('Get_CurrentReminder')?['ID']), ' kao poslat nije uspelo. Podsetnik može biti ponovo poslat.')`
2. Do not increment varFailedCount for this — email was attempted. Note in log.

#### 5b-v — Log per-reminder success and increment count

Add **Condition**: all email sends succeeded for this reminder.

If all sent:
1. Call LogEvent — Info/Success: `concat('Podsetnik "', outputs('Get_CurrentReminder')?['Title'], '" je poslat primaocem: ', string(length(outputs('Filter_ValidRecipients'))), '.')`
2. Set `varSentCount` = `add(variables('varSentCount'), 1)`

---

## Step 6 — Log overall result

After the Apply to each (or after the no-reminders path):

Add **Condition**: `equals(variables('varFailedCount'), 0)`

Yes: Call LogEvent — Info / Success
- message: `concat('Slanje podsetnika završeno. Poslato: ', string(variables('varSentCount')), ', Neuspešno: 0.')`

No: Call LogEvent — Warning / Failed
- message: `concat('Slanje podsetnika završeno. Poslato: ', string(variables('varSentCount')), ', Neuspešno: ', string(variables('varFailedCount')), '.')`

---

## Step 7 — Catch scope

Configure Run after: failed, timed out (on Try scope or on Get_DueReminders if not using a scope).

Call CF_DocCentralV3_LogEvent:
- eventType: `SendReminder` / severity: `Error` / status: `Failed`
- errorMessage: `result('Try')?[0]?['error']?['message']`
- correlationId: `variables('varCorrelationId')`
- message: `Neočekivana greška pri slanju podsetnika.`

Scheduled flows have no output. The Catch scope logs the failure and ends.

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_SendReminders`
- [ ] Trigger: Recurrence, daily at 08:00 UTC (or configured time)
- [ ] Concurrency Control set to 1 (singleton) in flow settings
- [ ] 4 variables initialized
- [ ] varTodayDate computed correctly (UTC or timezone-converted per client config)
- [ ] Log Started before Try scope
- [ ] Get_DueReminders filters by ReminderDate = today AND IsSent = false
- [ ] No reminders case: log Info, continue gracefully (no error)
- [ ] Apply to each iterates over due reminders
- [ ] Per-reminder scope allows per-item failure without stopping loop
- [ ] RecipientEmails parsed correctly per field type (Text or People)
- [ ] Empty recipient list: log Warning, still mark IsSent = true
- [ ] Send email inner loop: email failure is Warning (non-fatal)
- [ ] IsSent update always attempted (Run after: succeeded and failed)
- [ ] IsSent update failure: log Warning only, do not count as send failure
- [ ] varSentCount and varFailedCount tracked at reminder level
- [ ] Overall log after loop (Info or Warning by result)
- [ ] Catch scope: log Error
- [ ] All UNKNOWN internal column names resolved before building
- [ ] IsSent and SentAt columns created in Podsetnici list before building
- [ ] Flow inside DocCentralV3 solution
