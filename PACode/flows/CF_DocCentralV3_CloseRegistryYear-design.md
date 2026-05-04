# Flow design: CF_DocCentralV3_CloseRegistryYear

## Purpose

Closes the active registry year — an irreversible operation.
Verifies all preconditions before closing.
Creates a new registry book entry for the next year in App Config.
Updates the active year in App Config.
Logs the event.

**This operation can never be undone. No unlock mechanism exists or may be implemented.**

## Trigger type

Power Apps V2. Called directly from Canvas App (year closing screen).

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

```json
{
  "yearToClose": 2026,
  "initiatorEmail": "",
  "correlationId": ""
}
```

- `yearToClose`: the year being closed. Must match the currently active year in App Config.
- The flow re-validates this against App Config — it does not trust the Canvas App's year value alone.

## Output schema

Success:
```json
{
  "success": true,
  "message": "Godina je uspešno zaključena. Nova delovodna knjiga je kreirana za narednu godinu.",
  "closedYear": 2026,
  "newActiveYear": 2027,
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Zaključenje godine nije moguće.",
  "errorCode": "",
  "failedPrecondition": "",
  "correlationId": ""
}
```

## Preconditions (all must pass)

| # | Precondition | Failure code |
|---|---|---|
| 1 | `yearToClose` equals the active year in App Config | YEAR_NOT_ACTIVE |
| 2 | Active year is not already marked as closed in App Config | YEAR_ALREADY_CLOSED |
| 3 | Count of documents in Svi predmeti with Stanje ≠ `Arhivirano` for this year = 0 | DOCUMENTS_NOT_ALL_ARCHIVED |
| 4 | Count of items in RezervisaniBrojevi for this year = 0 | RESERVED_NUMBERS_EXIST |

All precondition checks are performed by the flow, not trusted from the Canvas App.
If any fails, the flow stops and returns the specific failure code and description.

## Flow steps

### Try scope

**Step 1 — Initialize variables**
- `varCorrelationId` = input or `guid()`
- `varActiveYear` (Integer) = 0
- `varNewYear` (Integer) = 0
- `varRegistryBookItemId` (Integer) = 0
- `varRegistryBookETag` (String) = empty

**Step 2 — Log Started**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `CloseRegistryYear`
- EventCategory: `YearClosing`
- Severity: `Info`
- Status: `Started`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Pokretanje zaključenja registarske godine ", string(yearToClose), ".")`

### Precondition checks

**Step 3 — Read active year and registry book from App Config**
Action: SharePoint — Get items from App Config filtered by active registry book config (UNKNOWN key).
Take top 1.

Store:
- `varActiveYear` = active year value
- `varRegistryBookItemId` = item ID
- `varRegistryBookETag` = item `@odata.etag`
- `varYearAlreadyClosed` = closed flag value (UNKNOWN column name)

**Step 4 — Precondition 1: year match**
If `yearToClose` ≠ `varActiveYear`:
Log Warning. Return failure `YEAR_NOT_ACTIVE`.

**Step 5 — Precondition 2: not already closed**
If `varYearAlreadyClosed = true`:
Log Warning. Return failure `YEAR_ALREADY_CLOSED`.

**Step 6 — Precondition 3: all documents archived**
Action: SharePoint — Get items from Svi predmeti
Filter: year = `yearToClose` AND `Stanje` ≠ `Arhivirano`
Top: 1

(Filter by year requires a year column or extracting year from filing date. UNKNOWN column name for year — assume `FilingYear` or derived from `FilingDate` field.)

If any item found:
- Log Warning (include count if retrievable).
- Return failure `DOCUMENTS_NOT_ALL_ARCHIVED`
  with `failedPrecondition` = "Postoje dokumenti koji nisu arhivirani."

**Step 7 — Precondition 4: no reserved numbers**
Action: SharePoint — Get items from RezervisaniBrojevi
Filter: year = `yearToClose`
Top: 1

(UNKNOWN column name for year on RezervisaniBrojevi items.)

If any item found:
- Log Warning.
- Return failure `RESERVED_NUMBERS_EXIST`
  with `failedPrecondition` = "Postoje rezervisani brojevi za ovu godinu."

### Close year and create new registry book

All preconditions passed. Proceed to close.

**Step 8 — Mark active year as closed in App Config (If-Match)**
Action: SharePoint HTTP PATCH on the active registry book App Config item
Header: `If-Match: <varRegistryBookETag>`

Fields to update (UNKNOWN internal column names):
- Closed flag = `true`
- ClosedAt = `utcNow()`
- ClosedBy = input `initiatorEmail`

If 412 Precondition Failed:
- Log Error (concurrent modification on the registry book config — should not happen in normal usage).
- Return failure `CONCURRENT_UPDATE`.

If any other error: return failure `CLOSE_YEAR_FAILED`.

**Step 9 — Calculate new year**
`varNewYear` = `yearToClose` + 1

**Step 10 — Create new registry book entry in App Config**
Action: SharePoint — Create item in App Config list

Fields for the new registry book (UNKNOWN internal column names and exact App Config structure):
- Category / Type = UNKNOWN (same category as existing registry book items)
- Active year = `varNewYear`
- Counter = 0 (or 1 depending on whether counter means "last used" or "next to use" — UNKNOWN)
- IsClosed = false
- Registry number format = copy from closing year's config

Store new item ID as `varNewRegistryBookItemId`.

**Step 11 — Update active year pointer in App Config**
If a separate "active year" pointer item exists in App Config (UNKNOWN — depends on App Config schema),
update it to `varNewYear`.

If the active year is determined by which registry book item is not closed, this step may be
implicit (the old book is now closed, the new book is open). UNKNOWN — to be confirmed from
App Config schema.

**Step 12 — Log Success**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `CloseRegistryYear`
- EventCategory: `YearClosing`
- Severity: `Info`
- Status: `Success`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: `concat("Godina ", string(yearToClose), " je zaključena. Nova aktivna godina: ", string(varNewYear), ".")`

**Step 13 — Return success response**
Include `closedYear` and `newActiveYear`.

### Catch scope

Call `CF_DocCentralV3_LogEvent`:
- EventType: `CloseRegistryYear`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `CLOSE_YEAR_FAILED`
- CorrelationId: `varCorrelationId`

Return failure response.

## Irreversibility guarantee

No step in this flow or any other flow may reverse the closing of a year.
The `varYearAlreadyClosed` flag (or equivalent App Config column) checked in Step 5
ensures that once closed, the flow refuses to process the same year again.
There is no admin override, no hidden parameter, and no backend endpoint that unlocks a closed year.
This constraint is enforced both in this flow and in `CF_DocCentralV3_CreateDocument`
(which checks that the active year is open before allowing registration).

## Error codes

| Code | Meaning |
|---|---|
| YEAR_NOT_ACTIVE | Input year does not match App Config active year |
| YEAR_ALREADY_CLOSED | Active year is already marked as closed |
| DOCUMENTS_NOT_ALL_ARCHIVED | One or more documents for this year are not Arhivirano |
| RESERVED_NUMBERS_EXIST | Reserved numbers still exist for this year |
| CONCURRENT_UPDATE | Registry book App Config item was modified concurrently |
| CLOSE_YEAR_FAILED | General failure during year closing operations |

## Open items

| Item | Status |
|---|---|
| App Config column for active year | UNKNOWN |
| App Config column for closed flag on registry book | UNKNOWN |
| App Config column for counter initial value (0 or 1) | UNKNOWN |
| Svi predmeti year column name for filtering | UNKNOWN |
| RezervisaniBrojevi year column name for filtering | UNKNOWN |
| Whether active year pointer is separate App Config item | UNKNOWN |
| Whether registry number format copies or resets | UNKNOWN |

## Canvas App pre-validation

The Canvas App year-closing screen should run read-only precondition checks before enabling
the submit button:
- Count of non-archived documents (from SharePoint read).
- Count of reserved numbers.

These are informational only — the flow re-validates authoritatively before executing.

## Test scenarios

| Scenario | Expected result |
|---|---|
| All conditions met, valid year | Year closed, new registry book created |
| Year does not match active year | YEAR_NOT_ACTIVE |
| Year already closed | YEAR_ALREADY_CLOSED |
| One document still Zavedeno | DOCUMENTS_NOT_ALL_ARCHIVED |
| One reserved number remains | RESERVED_NUMBERS_EXIST |
| Two admins submit simultaneously | Second gets CONCURRENT_UPDATE or YEAR_ALREADY_CLOSED |
| New registry book creation fails | CLOSE_YEAR_FAILED, year state rollback attempted |
