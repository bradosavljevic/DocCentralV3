# Implementation steps: CF_DocCentralV3_CloseRegistryYear

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- All documents for the active year are in Stanje = `Arhivirano` (enforced by precondition 3).
- RezervisaniBrojevi list is empty for the active year (enforced by precondition 4).
- App Config schema understood: registry book item structure, closed flag column, counter column, format string column.
- Internal column names for Svi predmeti year filter and RezervisaniBrojevi year filter must be resolved.
- ETag pattern understood — see `PACode/flows/CF_DocCentralV3_GenerateRegistryNumber/concurrency-etag-pattern.md`.
- Archive and year rules understood — see `PACode/flows/archive-year-rules.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_CloseRegistryYear`.
3. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| yearToClose | Number | `triggerBody()?['decimal_yearToClose']` |
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

Note: `yearToClose` is Number type — uses `decimal_` prefix in the trigger body.

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varYearToClose | Integer | `int(triggerBody()?['decimal_yearToClose'])` |
| varActiveYear | Integer | `0` |
| varNewYear | Integer | `0` |
| varRegistryBookItemId | Integer | `0` |
| varRegistryBookETag | String | `''` |
| varRequestDigest | String | `''` |

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `CloseRegistryYear` / eventCategory: `YearClosing`
- severity: `Info` / status: `Started`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Pokretanje zaključenja registarske godine ', string(variables('varYearToClose')), '.')`

---

## Step 5 — Try scope

### 5a — Read active registry book from App Config

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter: UNKNOWN — filter for registry book config item(s) where IsClosed = false (or by category key)
- Top: 1
- Order by: UNKNOWN (most recent year descending, or by creation date)

Rename: `Get_ActiveRegistryBook`

If result is empty (no active registry book found):
1. Call LogEvent — Warning.
2. Respond failure: errorCode = `YEAR_NOT_ACTIVE`, message = `Nema aktivne delovodne knjige u App Config.`
3. **Terminate Succeeded.**

Set variables from first result:
- `varActiveYear` = `int(first(body('Get_ActiveRegistryBook')?['value'])?['UNKNOWN_ActiveYear'])`
- `varRegistryBookItemId` = `int(first(body('Get_ActiveRegistryBook')?['value'])?['ID'])`
- `varRegistryBookETag` = `first(body('Get_ActiveRegistryBook')?['value'])?['@odata.etag']`

Note: The `@odata.etag` is available on items returned by the standard Get items connector action.
If not available, obtain it via a subsequent Get item call using `varRegistryBookItemId`.

Store the full item object for copying format string:
Add **Compose** — `Get_RegistryBookItem`:
```
first(body('Get_ActiveRegistryBook')?['value'])
```

### 5b — Precondition 1: year match

Add **Condition**: `not(equals(variables('varActiveYear'), variables('varYearToClose')))`

Yes:
1. Call LogEvent — Warning: `concat('Pokušaj zaključenja godine ', string(variables('varYearToClose')), ', ali aktivna godina je ', string(variables('varActiveYear')), '.')`
2. Respond failure: errorCode = `YEAR_NOT_ACTIVE`, failedPrecondition = `concat('Aktivna godina u sistemu je ', string(variables('varActiveYear')), '.')`
3. **Terminate Succeeded.**

### 5c — Precondition 2: year not already closed

Add **Condition**: `equals(outputs('Get_RegistryBookItem')?['UNKNOWN_IsClosed'], true)`

Yes:
1. Call LogEvent — Warning: `concat('Godina ', string(variables('varYearToClose')), ' je već zaključena.')`
2. Respond failure: errorCode = `YEAR_ALREADY_CLOSED`, failedPrecondition = `Godina je već zaključena.`
3. **Terminate Succeeded.**

### 5d — Precondition 3: all documents archived

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- Filter: `UNKNOWN_Year eq <varYearToClose> and UNKNOWN_Stanje ne 'Arhivirano'`
  - (UNKNOWN: year column name and Stanje internal name — must be resolved before building)
- Top: 1

Rename: `Get_NonArchivedDocs`

Add **Condition**: `greater(length(body('Get_NonArchivedDocs')?['value']), 0)`

Yes:
1. Call LogEvent — Warning: `concat('Postoje dokumenti za godinu ', string(variables('varYearToClose')), ' koji nisu arhivirani.')`
2. Respond failure: errorCode = `DOCUMENTS_NOT_ALL_ARCHIVED`, failedPrecondition = `Postoje dokumenti koji nisu arhivirani.`
3. **Terminate Succeeded.**

### 5e — Precondition 4: no reserved numbers

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')`
- Filter: `UNKNOWN_Year eq <varYearToClose>`
  - (UNKNOWN: year column name on RezervisaniBrojevi — must be resolved)
- Top: 1

Rename: `Get_ReservedNumbers`

Add **Condition**: `greater(length(body('Get_ReservedNumbers')?['value']), 0)`

Yes:
1. Call LogEvent — Warning: `concat('Postoje rezervisani brojevi za godinu ', string(variables('varYearToClose')), '.')`
2. Respond failure: errorCode = `RESERVED_NUMBERS_EXIST`, failedPrecondition = `Postoje rezervisani brojevi za ovu godinu.`
3. **Terminate Succeeded.**

---

### All preconditions passed — proceed to close

### 5f — Obtain request digest

Add **SharePoint — Send an HTTP Request**:
- Method: POST
- Uri: `_api/contextinfo`
- Headers: `Accept: application/json;odata=verbose`

Rename: `Get_RequestDigest`

Add **Compose** — `Extract_RequestDigest`:
```
body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']
```

Set `varRequestDigest` = `outputs('Extract_RequestDigest')`

### 5g — PATCH App Config: mark year as closed (If-Match)

Add **SharePoint — Send an HTTP Request** — PATCH:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Method: POST
- Uri: `concat("_api/web/lists/GetByTitle('", @parameters('gpdoccen_EV_DocCentralV3_lstAppConfig'), "')/items(", variables('varRegistryBookItemId'), ")")`
- Headers:
  - `Accept`: `application/json;odata=verbose`
  - `Content-Type`: `application/json;odata=verbose`
  - `IF-MATCH`: `variables('varRegistryBookETag')`
  - `X-HTTP-Method`: `MERGE`
  - `X-RequestDigest`: `variables('varRequestDigest')`
- Body (all UNKNOWN internal column names):
```json
{
  "__metadata": { "type": "SP.Data.AppConfigListItem" },
  "UNKNOWN_IsClosed": true,
  "UNKNOWN_ClosedAt": "<utcNow()>",
  "UNKNOWN_ClosedBy": "<triggerBody()['text_initiatorEmail']>"
}
```

Rename: `PATCH_AppConfig_CloseYear`

**412 handling:**

Add **Condition**: `equals(outputs('PATCH_AppConfig_CloseYear')?['statusCode'], 412)`

Yes:
1. Call LogEvent — Error: `concat('Greška pri zaključenju: konkurentna izmena App Config stavke za godinu ', string(variables('varYearToClose')), '.')`
2. Respond failure: errorCode = `CONCURRENT_UPDATE`, message = `Zaključenje nije uspelo zbog konkurentne izmene.`
3. **Terminate Succeeded.**

**Other error handling:**

Add **Condition**: `not(or(equals(...statusCode, 200), equals(...statusCode, 204)))`

Yes:
1. Call LogEvent — Error.
2. Respond failure: errorCode = `CLOSE_YEAR_FAILED`.
3. **Terminate Succeeded.**

### 5h — Calculate new year

Set `varNewYear` = `add(variables('varYearToClose'), 1)`

### 5i — Create new registry book entry in App Config

Add **SharePoint — Create item**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Fields (all UNKNOWN internal column names — copy structure from closing year item):
  - Category / Type field = UNKNOWN (same value as existing registry book items)
  - UNKNOWN_ActiveYear = `variables('varNewYear')`
  - UNKNOWN_Counter = `0` (UNKNOWN — confirm with GenerateRegistryNumber whether counter starts at 0 or 1)
  - UNKNOWN_IsClosed = `false`
  - UNKNOWN_FormatString = `outputs('Get_RegistryBookItem')?['UNKNOWN_FormatString']` (copy from closing year)
  - Any other registry book config fields — UNKNOWN

Rename: `Create_NewRegistryBook`

If creation fails:
1. Call LogEvent — Error: `CLOSE_YEAR_FAILED` — new registry book not created; year was already marked closed.
2. This is a partial-complete state — the year is closed but no new book exists.
   Log as Critical. Return CLOSE_YEAR_FAILED.
   **Note:** There is no rollback for the year-close PATCH. This is an admin-recoverable state.
3. **Terminate Succeeded.**

### 5j — Update active year pointer (if separate item exists in App Config)

UNKNOWN — depends on App Config schema.

If the active year is determined by which registry book item is not closed (IsClosed = false), then
creating the new registry book (Step 5i) is sufficient — no additional PATCH needed.

If there is a separate "active year" config item (e.g. a row with Key = `ActiveYear`), update it:

Add **SharePoint — Send an HTTP Request** — PATCH on that item:
- UNKNOWN_Value = `string(variables('varNewYear'))`
- If-Match with that item's ETag (read it in Step 5a along with the registry book).

Mark as UNKNOWN — skip this step if App Config schema uses IsClosed flag pattern.

### 5k — Log Success

Call CF_DocCentralV3_LogEvent:
- eventType: `CloseRegistryYear` / severity: `Info` / status: `Success`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Godina ', string(variables('varYearToClose')), ' je zaključena. Nova aktivna godina: ', string(variables('varNewYear')), '.')`

### 5l — Respond success

**Respond to a PowerApp or flow** with 5 outputs:

| Output | Value |
|---|---|
| success | `true` |
| message | `Godina je uspešno zaključena. Nova delovodna knjiga je kreirana za narednu godinu.` |
| closedYear | `variables('varYearToClose')` |
| newActiveYear | `variables('varNewYear')` |
| correlationId | `variables('varCorrelationId')` |
| errorCode | `''` |
| failedPrecondition | `''` |

---

## Step 6 — Catch scope

Configure Run after: failed, timed out (on the Try scope).

Call CF_DocCentralV3_LogEvent:
- eventType: `CloseRegistryYear` / severity: `Error` / status: `Failed`
- errorCode: `CLOSE_YEAR_FAILED`
- errorMessage: `result('Try')?[0]?['error']?['message']`
- correlationId: `variables('varCorrelationId')`

Respond failure:
- success: false
- message: `Zaključenje godine nije uspelo.`
- errorCode: `CLOSE_YEAR_FAILED`
- failedPrecondition: `''`
- correlationId: `variables('varCorrelationId')`
- closedYear: `0`
- newActiveYear: `0`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_CloseRegistryYear`
- [ ] Trigger: Power Apps (V2) with 3 inputs (yearToClose as Number)
- [ ] All 7 variables initialized; varYearToClose = int(decimal_yearToClose)
- [ ] Log Started before preconditions
- [ ] Precondition 1: year match (validates against App Config, not trusted from input)
- [ ] Precondition 2: year not already closed (IsClosed flag check)
- [ ] Precondition 3: Svi predmeti filtered by year AND Stanje ≠ Arhivirano, Top 1
- [ ] Precondition 4: RezervisaniBrojevi filtered by year, Top 1
- [ ] All precondition failures use Terminate Succeeded (not Terminate Failed)
- [ ] Request digest obtained before PATCH
- [ ] PATCH App Config uses IF-MATCH with varRegistryBookETag
- [ ] 412: log Error, return CONCURRENT_UPDATE, Terminate Succeeded
- [ ] New registry book created in App Config with correct fields copied
- [ ] Active year pointer updated if separate item (UNKNOWN — conditional)
- [ ] Log Success before Respond
- [ ] Catch scope: log Error, respond failure
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow inside DocCentralV3 solution
