# Flow design: CF_DocCentralV3_UseReservedNumber

## Purpose

Validates and consumes a reserved registry number from the `RezervisaniBrojevi` list.
Called as a child flow by `CF_DocCentralV3_CreateDocument` when the user selects a reserved number.
The reserved number is removed from the list only after the parent document has been successfully created.

## Trigger type

Child flow (called by `CF_DocCentralV3_CreateDocument`). Not called directly from Canvas App.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

## Input schema

```json
{
  "reservedNumberId": 0,
  "requestedYear": 2026,
  "initiatorEmail": "",
  "correlationId": ""
}
```

`reservedNumberId` is the SharePoint item ID from `RezervisaniBrojevi`.
`requestedYear` is the year the user is filing the document into (must match the reserved number's year).

## Output schema

Success — number validated, ready to use (not yet deleted):
```json
{
  "success": true,
  "delovodniBroj": "15/2026",
  "reservedNumberId": 0,
  "filingDateSuggestion": "2026-01-15",
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Rezervisani broj nije validan ili je već iskorišćen.",
  "errorCode": "RESERVED_NUMBER_INVALID",
  "correlationId": ""
}
```

## Reserved number validation rules

All rules must be checked before the number is used.

| Rule | Description |
|---|---|
| Number exists | The item with `reservedNumberId` must exist in `RezervisaniBrojevi`. |
| Year match | The reserved number's year must match `requestedYear` (active year from App Config). |
| Not already used | No existing document in `Svi predmeti` may already have this `DelovodniBroj`. |
| Active year check | The `requestedYear` must match the active year in App Config. Cannot use a reserved number from a prior or future year. |

## Flow steps

### Initialization

**Step 1 — Initialize variables**
- `varCorrelationId` = input `correlationId` if not empty, else `guid()`
- `varDelovodniBroj` (String) = empty
- `varReservedYear` (Integer) = 0

**Step 2 — Log Started event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `UseReservedNumber`
- EventCategory: `ReservedNumber`
- Severity: `Info`
- Status: `Started`
- ReservedNumberId: input `reservedNumberId`
- UserEmail: input `initiatorEmail`
- CorrelationId: `varCorrelationId`
- Message: "Pokretanje korišćenja rezervisanog broja."

### Read and validate reserved number

**Step 3 — Get reserved number item**
Action: SharePoint — Get item
List: `EV_DocCentralV3_lstRezervisaniBrojevi`
ID: input `reservedNumberId`

If item not found (SharePoint 404): log error. Return failure with code `RESERVED_NUMBER_NOT_FOUND`.

Store:
- `varDelovodniBroj` = the `DelovodniBroj` column value from the item
- `varReservedYear` = the year column value from the item (column name UNKNOWN — to be confirmed from list schema)
- `varETag` = item `@odata.etag` (held in case a soft-lock update is needed in future)

**Step 4 — Validate year**
Compare `varReservedYear` with input `requestedYear`.
If mismatch: log warning. Return failure with code `RESERVED_NUMBER_YEAR_MISMATCH`.

**Step 5 — Read active year from App Config**
Read active year from App Config (UNKNOWN key).
If `requestedYear` does not equal the active year in App Config: log warning. Return failure with code `YEAR_NOT_ACTIVE`.

**Step 6 — Check for existing document with same DelovodniBroj**
Action: SharePoint — Get items
List: `EV_DocCentralV3_lstSviPredmeti`
Filter: `DelovodniBroj eq '<varDelovodniBroj>'`
Top: 1

If any item found: the number has already been used. Log error with Severity Critical.
Return failure with code `RESERVED_NUMBER_ALREADY_USED`.

### Return validated number (do not delete yet)

**Step 7 — Return success**
At this point the number is validated but the reserved number item is NOT deleted.
Deletion happens in `CF_DocCentralV3_CreateDocument` only after the document has been successfully created.

Return success response with the validated `DelovodniBroj`.

## Deletion responsibility

The delete step is deliberately separated from this flow.
`CF_DocCentralV3_CreateDocument` calls this flow first, then creates the document, and only then deletes the reserved number item.

This ensures:
- If document creation fails, the reserved number is not consumed and remains available.
- Deletion is atomic to the document creation outcome.

Delete action when called from parent flow:
Action: SharePoint — Delete item
List: `EV_DocCentralV3_lstRezervisaniBrojevi`
ID: input `reservedNumberId`

After deletion, the parent flow logs:
- EventType: `UseReservedNumber`
- Status: `Success`
- Message: `concat("Rezervisani broj ", varDelovodniBroj, " je iskorišćen i uklonjen.")`

## Error handling

**Catch scope:**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `UseReservedNumber`
- Severity: `Error`
- Status: `Failed`
- ErrorMessage: error details
- CorrelationId: `varCorrelationId`

Return failure response.

## Error codes

| Code | Meaning |
|---|---|
| RESERVED_NUMBER_NOT_FOUND | Item does not exist in RezervisaniBrojevi |
| RESERVED_NUMBER_YEAR_MISMATCH | Item year does not match requested year |
| YEAR_NOT_ACTIVE | Requested year is not the active registry year |
| RESERVED_NUMBER_ALREADY_USED | A document with the same DelovodniBroj already exists |

## Open items

| Item | Status |
|---|---|
| Year column name in RezervisaniBrojevi | UNKNOWN — must be confirmed from list schema |
| Filing date column name in RezervisaniBrojevi | UNKNOWN |
| Additional validation rules per ProcesConfig | UNKNOWN |

## Audit events summary

| When | EventType | Status | Severity |
|---|---|---|---|
| Flow starts | UseReservedNumber | Started | Info |
| Number not found | UseReservedNumber | Failed | Error |
| Year mismatch | UseReservedNumber | Failed | Warning |
| Already used | UseReservedNumber | Failed | Critical |
| Validated successfully | UseReservedNumber | Success | Info |
| Deleted after document creation (from parent) | UseReservedNumber | Success | Info |

## Permission

Runs under service account via `CR_DocCentralV3_SharePoint`.
Service account must have Read access to `RezervisaniBrojevi` list.
Delete is executed by parent flow under same service account.

## Test scenarios

| Scenario | Expected result |
|---|---|
| Valid reserved number, current year | Success response with DelovodniBroj |
| Reserved number ID does not exist | Failure: RESERVED_NUMBER_NOT_FOUND |
| Reserved number is from previous year | Failure: RESERVED_NUMBER_YEAR_MISMATCH |
| Number already used in Svi predmeti | Failure: RESERVED_NUMBER_ALREADY_USED, Critical audit log |
| Document creation fails after validation | Reserved number NOT deleted, remains available |
| Document creation succeeds | Reserved number deleted by parent flow |
