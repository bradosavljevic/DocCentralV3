# Implementation steps: CF_DocCentralV3_UseReservedNumber

## Prerequisites

- CF_DocCentralV3_LogEvent is built and published in the DocCentralV3 solution.
- RezervisaniBrojevi SharePoint list exists with at least a DelovodniBroj column and a year column (name UNKNOWN — confirm before building).
- Svi predmeti SharePoint list exists.
- App Config SharePoint list exists with an active year entry.
- AuditLog SharePoint list exists.
- Connection reference CR_DocCentralV3_SharePoint is connected.
- Environment variables registered: EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstRezervisaniBrojevi, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAuditLog.

---

## Step 1 — Create the flow

1. Open the DocCentralV3 solution.
2. New → Cloud flow → Instant cloud flow.
3. Name: `CF_DocCentralV3_UseReservedNumber`.
4. Trigger: select Manually trigger a flow as a placeholder — replace it in the next step.

---

## Step 2 — Replace trigger with Power Apps (V2)

This flow is called as a child flow by CF_DocCentralV3_CreateDocument.

1. Delete the placeholder trigger.
2. Add trigger: **Power Apps (V2)**.
3. Add inputs:

| Input name | Type |
|---|---|
| reservedNumberId | Number |
| requestedYear | Number |
| initiatorEmail | Text |
| correlationId | Text |

Internal references in expressions:
- `triggerBody()?['decimal_reservedNumberId']`
- `triggerBody()?['decimal_requestedYear']`
- `triggerBody()?['text_initiatorEmail']`
- `triggerBody()?['text_correlationId']`

Always verify actual internal names in the expression editor after adding inputs.
The `decimal_` prefix is typical for Number inputs in Power Apps V2 triggers.

---

## Step 3 — Initialize variables

Add **Initialize variable** actions before the Try scope:

| Variable name | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varDelovodniBroj | String | `''` |
| varReservedYear | Integer | `0` |
| varETag | String | `''` |
| varActiveYear | Integer | `0` |

Initialize varCorrelationId first — all LogEvent calls depend on it.

---

## Step 4 — Log Started

Add **Run a Child Flow** action:
- Child flow: `CF_DocCentralV3_LogEvent`

| Parameter | Value |
|---|---|
| eventType | `UseReservedNumber` |
| eventCategory | `ReservedNumber` |
| severity | `Info` |
| status | `Started` |
| correlationId | `variables('varCorrelationId')` |
| flowName | `CF_DocCentralV3_UseReservedNumber` |
| flowRunId | `workflow().run.name` |
| source | `CloudFlow` |
| userEmail | `triggerBody()?['text_initiatorEmail']` |
| reservedNumberId | `int(triggerBody()?['decimal_reservedNumberId'])` |
| message | `Pokretanje korišćenja rezervisanog broja.` |

---

## Step 5 — Try scope

Add a **Scope** action. Name it `Try`. All validation logic goes inside.

### 5a — Get reserved number item from RezervisaniBrojevi

Add **SharePoint — Get item**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')`
- ID: `int(triggerBody()?['decimal_reservedNumberId'])`

**Handle item not found (404):**
The Get item action fails with a 404 if the item does not exist.
Add a **Condition** after Get item that checks whether the action status is an error:
- Expression: `outputs('Get_item')?['statusCode']`
- Operator: is equal to
- Value: `404`

If 404 (Yes branch):
1. Call CF_DocCentralV3_LogEvent:
   - severity: `Error` / status: `Failed` / errorCode: `RESERVED_NUMBER_NOT_FOUND`
   - message: `Rezervisani broj nije pronađen u listi.`
2. Add **Respond to a PowerApp or flow** — failure response with errorCode `RESERVED_NUMBER_NOT_FOUND`.
3. Add **Terminate** action — Status: Succeeded (to exit the Try scope cleanly after responding).

If item found (No branch — item exists, continue):

Extract values from the item:
- Set varDelovodniBroj = `outputs('Get_item')?['body']?['DelovodniBroj']`
  (Replace `DelovodniBroj` with the actual internal column name from the list — UNKNOWN)
- Set varReservedYear = `int(outputs('Get_item')?['body']?['UNKNOWN_YearColumn'])`
  (Replace UNKNOWN_YearColumn with actual internal name — see `reserved-number-field-mapping.md`)
- Set varETag = `outputs('Get_item')?['body']?['@odata.etag']`

### 5b — Validate year: reserved number year vs requested year

Add **Condition**:
- Left: `variables('varReservedYear')`
- Operator: is not equal to
- Right: `int(triggerBody()?['decimal_requestedYear'])`

If mismatch (Yes branch):
1. Call CF_DocCentralV3_LogEvent:
   - severity: `Warning` / status: `Failed` / errorCode: `RESERVED_NUMBER_YEAR_MISMATCH`
   - message: `concat('Godina rezervisanog broja (', string(variables('varReservedYear')), ') ne odgovara traženo godini (', string(int(triggerBody()?[''decimal_requestedYear''])), ').')`
2. Add **Respond** — failure response with errorCode `RESERVED_NUMBER_YEAR_MISMATCH`.
3. Add **Terminate** — Status: Succeeded.

If year matches (No branch — continue):

### 5c — Read active year from App Config

Add **SharePoint — Get items**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter Query: UNKNOWN — filter to the active year App Config row. See `reserved-number-field-mapping.md`.
- Top Count: `1`

Extract active year:
- Set varActiveYear = `int(first(body('Get_items')?['value'])?['UNKNOWN_ActiveYearColumn'])`

Add **Condition**:
- Left: `int(triggerBody()?['decimal_requestedYear'])`
- Operator: is not equal to
- Right: `variables('varActiveYear')`

If mismatch (Yes branch):
1. Call CF_DocCentralV3_LogEvent:
   - severity: `Warning` / status: `Failed` / errorCode: `YEAR_NOT_ACTIVE`
   - message: `concat('Tražena godina (', string(int(triggerBody()?[''decimal_requestedYear''])), ') nije aktivna godina zavođenja.')`
2. Add **Respond** — failure response with errorCode `YEAR_NOT_ACTIVE`.
3. Add **Terminate** — Status: Succeeded.

### 5d — Check for duplicate: does Svi predmeti already have this DelovodniBroj?

Add **SharePoint — Get items**:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- Filter Query: `DelovodniBroj eq '@{variables('varDelovodniBroj')}'`
  (Replace `DelovodniBroj` with the actual internal column name from Svi predmeti — UNKNOWN)
- Top Count: `1`

Rename this action: `Get_SviPredmeti_DuplicateCheck`

Add **Condition**:
- Left: `length(body('Get_SviPredmeti_DuplicateCheck')?['value'])`
- Operator: greater than
- Right: `0`

If duplicate found (Yes branch):
1. Call CF_DocCentralV3_LogEvent:
   - severity: `Critical` / status: `Failed` / errorCode: `RESERVED_NUMBER_ALREADY_USED`
   - message: `concat('Delovodni broj ', variables('varDelovodniBroj'), ' već postoji u Svi predmeti.')`
2. Add **Respond** — failure response with errorCode `RESERVED_NUMBER_ALREADY_USED`.
3. Add **Terminate** — Status: Succeeded.

### 5e — Return success (number validated, not yet deleted)

Add **Respond to a PowerApp or flow** — success response.

| Output | Value |
|---|---|
| success | `true` |
| delovodniBroj | `variables('varDelovodniBroj')` |
| reservedNumberId | `int(triggerBody()?['decimal_reservedNumberId'])` |
| filingDateSuggestion | `outputs('Get_item')?['body']?['UNKNOWN_FilingDateColumn']` — UNKNOWN, see field mapping |
| correlationId | `variables('varCorrelationId')` |
| message | `''` |
| errorCode | `''` |

**Note:** The reserved number item is NOT deleted here. CF_DocCentralV3_CreateDocument
performs the deletion after successful document creation.

---

## Step 6 — Catch scope

Add a second **Scope** action after Try. Name it `Catch`.

Configure Run after:
- Uncheck `is successful`
- Check `has failed`
- Check `has timed out`

Inside Catch:
1. Call CF_DocCentralV3_LogEvent:
   - eventType: `PowerAutomateError`
   - severity: `Error`
   - status: `Failed`
   - correlationId: `variables('varCorrelationId')`
   - message: `Neočekivana greška pri korišćenju rezervisanog broja.`
   - errorCode: `RESERVED_NUMBER_VALIDATION_FAILED`
   - errorMessage: `result('Try')?[0]?['error']?['message']`
2. Add **Respond to a PowerApp or flow** — failure response.

---

## Step 7 — Post-scope success/failure routing

After Try and Catch scopes, add **Condition**:
- Left: `result('Try')?[0]?['status']`
- Operator: is equal to
- Right: `Succeeded`

**Yes branch:**
- Log validated successfully:
  - eventType: `UseReservedNumber` / severity: `Info` / status: `Success`
  - message: `concat('Rezervisani broj ', variables('varDelovodniBroj'), ' je validiran i spreman za korišćenje.')`
  - Note: this log entry marks validation success; the UseReservedNumber/Success/deletion log is written by CF_DocCentralV3_CreateDocument after deletion.

**No branch:** Catch scope already handled logging and respond. No additional action needed.

---

## Step 8 — Save and verify

1. Save the flow.
2. Verify it is inside the DocCentralV3 solution.
3. Run TC-001 through TC-004 before integrating with CF_DocCentralV3_CreateDocument.
4. Description: `"v1.0 — initial implementation. CF_DocCentralV3_UseReservedNumber. DocCentralV3 solution."`

---

## Deletion responsibility reminder

This flow does NOT delete the reserved number item. Deletion is the responsibility of
CF_DocCentralV3_CreateDocument, executed only after the document item in Svi predmeti
has been successfully created.

When CF_DocCentralV3_CreateDocument performs the deletion:
```
Action: SharePoint — Delete item
List: @parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')
ID: <reservedNumberId from caller input>
```

After deletion, CF_DocCentralV3_CreateDocument logs:
- eventType: `UseReservedNumber` / status: `Success` / severity: `Info`
- message: `concat('Rezervisani broj ', varDelovodniBroj, ' je iskorišćen i uklonjen.')`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_UseReservedNumber`
- [ ] Trigger is Power Apps (V2) with 4 inputs: reservedNumberId (Number), requestedYear (Number), initiatorEmail (Text), correlationId (Text)
- [ ] All 5 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Log Started called before Try scope
- [ ] Get item uses CR_DocCentralV3_SharePoint and EV expression for list name
- [ ] 404 handled: RESERVED_NUMBER_NOT_FOUND failure response returned
- [ ] varDelovodniBroj and varReservedYear populated from item fields
- [ ] Year validation: varReservedYear vs requestedYear — RESERVED_NUMBER_YEAR_MISMATCH on mismatch
- [ ] Active year validation from App Config — YEAR_NOT_ACTIVE on mismatch
- [ ] Duplicate check in Svi predmeti — RESERVED_NUMBER_ALREADY_USED with Critical log
- [ ] Success respond does NOT delete the reserved number item
- [ ] Catch scope runs on failed/timed out only
- [ ] Catch scope calls LogEvent (PowerAutomateError/Error/Failed)
- [ ] Catch scope does NOT call CF_DocCentralV3_UseReservedNumber
- [ ] All 7 output parameters in both Respond actions
- [ ] All SharePoint actions use CR_DocCentralV3_SharePoint
- [ ] All EV references use @parameters() syntax
- [ ] Flow is inside DocCentralV3 solution
