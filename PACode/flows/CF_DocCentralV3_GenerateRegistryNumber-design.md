# Flow design: CF_DocCentralV3_GenerateRegistryNumber

## Purpose

Generates the next unique registry number (`DelovodniBroj`) for the active registry book.
Uses optimistic locking (ETag / If-Match) to guarantee uniqueness under concurrent execution.
This flow is called as a child flow by `CF_DocCentralV3_CreateDocument`.

## Trigger type

Child flow (instantiated by parent flow). Not called directly from Canvas App.

## Connection references used

- `gpdoccen_CR_DocCentralV3_SharePoint`

## Environment variables used

- `gpdoccen_EV_DocCentralV3_SharePointSite`
- `gpdoccen_EV_DocCentralV3_lstAppConfig`
- `gpdoccen_EV_DocCentralV3_lstSviPredmeti`
- `gpdoccen_EV_DocCentralV3_lstAuditLog`

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
  "delovodniBroj": "42/2026",
  "counterValue": 42,
  "activeYear": 2026,
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Nije moguće generisati delovodni broj.",
  "errorCode": "REGISTRY_NUMBER_GENERATION_FAILED",
  "correlationId": ""
}
```

## App Config dependencies

The following App Config keys are required. Exact key names are UNKNOWN until AppConfig.csv is provided.

| Purpose | App Config key | Status |
|---|---|---|
| Active registry year | UNKNOWN | UNKNOWN |
| Active registry book item ID | UNKNOWN | UNKNOWN |
| Current counter value (next number) | UNKNOWN | UNKNOWN |
| Registry number format string | UNKNOWN | UNKNOWN |
| Max retry attempts | UNKNOWN | UNKNOWN |

The flow must read these values dynamically from App Config. No hardcoded values.

## Flow steps

### Initialization

**Step 1 — Initialize variables**
- `varRetryCount` (Integer) = 0
- `varMaxRetries` (Integer) = read from App Config (UNKNOWN key, default: 5 if not configured)
- `varCorrelationId` (String) = input correlationId if provided, else `guid()`
- `varSuccess` (Boolean) = false
- `varDelovodniBroj` (String) = empty
- `varCounterValue` (Integer) = 0

**Step 2 — Log Started event**
Call `CF_DocCentralV3_LogEvent` child flow:
- EventType: `GenerateRegistryNumber`
- EventCategory: `RegistryNumber`
- Severity: `Info`
- Status: `Started`
- CorrelationId: `varCorrelationId`
- UserEmail: input `initiatorEmail`
- Message: "Pokretanje generisanja delovodnog broja."

### Read App Config

**Step 3 — Read active registry book from App Config**
Action: SharePoint — Get items
List: `EV_DocCentralV3_lstAppConfig`
Filter: query for the active registry book item (filter by UNKNOWN key for active registry book).
Take top 1 result.

Store:
- `varRegistryBookItemId` = item ID
- `varCurrentCounter` = current counter value column value
- `varActiveYear` = active year column value
- `varRegistryFormat` = format string column value
- `varETag` = item `@odata.etag` value (required for optimistic locking)

If no active registry book found: fail immediately. Log error. Return failure response.

### Retry loop (optimistic locking)

**Step 4 — Do Until loop**
Condition to exit: `varSuccess = true` OR `varRetryCount >= varMaxRetries`

Inside loop:

**Step 4a — Calculate next number**
- `varNextCounter` = `varCurrentCounter` + 1
- Format `varDelovodniBroj` using format string from App Config.
  Example: if format is `{counter}/{year}`, result is `42/2026`.
  Formatting applied via Power Automate expression (e.g. `concat(string(varNextCounter), '/', string(varActiveYear))`).

**Step 4b — Attempt atomic counter update (If-Match)**
Action: SharePoint — Send an HTTP request (to update the App Config item).
Method: PATCH
URI: `<SharePointSite>/_api/web/lists/getByTitle('<AppConfigList>')/items(<varRegistryBookItemId>)`
Headers:
  - `If-Match: <varETag>`
  - `Content-Type: application/json;odata=verbose`
Body: JSON patch with the updated counter column set to `varNextCounter`.

**Step 4c — Evaluate result**
- If HTTP status 200: update succeeded. The number is reserved. Set `varCounterValue = varNextCounter`. Set `varSuccess = true`. Exit loop.
- If HTTP status 412 (Precondition Failed): concurrent conflict. Go to Step 4d.
- Any other error status: go to error handling (Step 6).

**Step 4d — Handle conflict (412)**
- Increment `varRetryCount` by 1.
- Log Retried event to AuditLog:
  - EventType: `GenerateRegistryNumber`
  - Severity: `Warning`
  - Status: `Retried`
  - Message: `concat("Konflikt pri generisanju broja, pokušaj ", string(varRetryCount), " od ", string(varMaxRetries))`
- If `varRetryCount < varMaxRetries`: wait a random delay (use `rand(500, 1500)` ms via Delay action), then re-read the App Config item fresh (repeat Step 3 logic inside loop) to get new ETag and current counter value. Continue loop.
- If `varRetryCount >= varMaxRetries`: exit loop (condition catches this). Proceed to Step 6 (error handling).

### Uniqueness safety check

**Step 5 — Verify no duplicate in Svi predmeti**
Action: SharePoint — Get items
List: `EV_DocCentralV3_lstSviPredmeti`
Filter: `DelovodniBroj eq '<varDelovodniBroj>'`
Top: 1

If any item found: log Critical error. Return failure response (do not proceed to create document).
If no item found: proceed.

### Success path

**Step 6 — Log Success event**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateRegistryNumber`
- Severity: `Info`
- Status: `Success`
- CorrelationId: `varCorrelationId`
- DelovodniBroj: `varDelovodniBroj`
- Message: `concat("Delovodni broj ", varDelovodniBroj, " je uspešno generisan.")`

**Step 7 — Return success response**
```json
{
  "success": true,
  "delovodniBroj": "<varDelovodniBroj>",
  "counterValue": "<varCounterValue>",
  "activeYear": "<varActiveYear>",
  "correlationId": "<varCorrelationId>"
}
```

### Error handling

**Catch scope — retry limit exceeded or non-conflict error**
Call `CF_DocCentralV3_LogEvent`:
- EventType: `GenerateRegistryNumber`
- Severity: `Error`
- Status: `Failed`
- ErrorCode: `REGISTRY_NUMBER_GENERATION_FAILED`
- ErrorMessage: details from the failed action
- CorrelationId: `varCorrelationId`

Return failure response:
```json
{
  "success": false,
  "message": "Nije moguće generisati delovodni broj.",
  "errorCode": "REGISTRY_NUMBER_GENERATION_FAILED",
  "correlationId": "<varCorrelationId>"
}
```

## Error codes

| Code | Meaning |
|---|---|
| REGISTRY_NUMBER_GENERATION_FAILED | General failure — retry limit exceeded or unexpected error |
| REGISTRY_NUMBER_DUPLICATE | Duplicate found in Svi predmeti (should never occur under correct locking) |
| REGISTRY_BOOK_NOT_FOUND | No active registry book found in App Config |

## Audit events summary

| When | EventType | Status | Severity |
|---|---|---|---|
| Flow starts | GenerateRegistryNumber | Started | Info |
| Conflict, retry | GenerateRegistryNumber | Retried | Warning |
| Success | GenerateRegistryNumber | Success | Info |
| Failure | GenerateRegistryNumber | Failed | Error |
| Duplicate detected | GenerateRegistryNumber | Failed | Critical |

## Permission

Runs under service account via `CR_DocCentralV3_SharePoint`.
Service account must have Write access to App Config list to update the counter.

## Test scenarios

| Scenario | Expected result |
|---|---|
| Single user, first document of the year | Counter reads 1, returns 1/2026, counter updated to 1 |
| Two concurrent flows, same counter value | One succeeds, one retries and gets next number |
| Retry limit exceeded | Failure logged, error response returned |
| No active registry book in App Config | Immediate failure, error logged |
| Duplicate found in Svi predmeti | Critical log entry, failure response, document not created |
