# Implementation steps: CF_DocCentralV3_GenerateRegistryNumber

## Prerequisites

- CF_DocCentralV3_LogEvent is built and published in the DocCentralV3 solution.
- App Config SharePoint list exists with a registry book item (active year, counter, format string).
- Svi predmeti SharePoint list exists.
- AuditLog SharePoint list exists.
- Connection reference CR_DocCentralV3_SharePoint is connected.
- Environment variables are registered: EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAuditLog.

---

## Step 1 — Create the flow

1. Open the DocCentralV3 solution.
2. New → Cloud flow → **Instant cloud flow**.
3. Name: `CF_DocCentralV3_GenerateRegistryNumber`.
4. Trigger: select **Manually trigger a flow** as a placeholder — you will replace it next.

---

## Step 2 — Replace trigger with Power Apps (V2) — child flow pattern

This flow is called as a **child flow**, not directly from Power Apps. The trigger must be **Power Apps (V2)** to support the Run a Child Flow action from parent flows.

1. Delete the placeholder trigger.
2. Add trigger: **Power Apps (V2)**.
3. Add inputs (two only):

| Input name | Type |
|---|---|
| initiatorEmail | Text |
| correlationId | Text |

Internal references in expressions:
- `triggerBody()?['text_initiatorEmail']`
- `triggerBody()?['text_correlationId']`

---

## Step 3 — Initialize variables

Add **Initialize variable** actions (before the Try scope):

| Variable name | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varRetryCount | Integer | `0` |
| varMaxRetries | Integer | `5` (default; overridden from App Config in Step 5) |
| varSuccess | Boolean | `false` |
| varDelovodniBroj | String | `''` |
| varCounterValue | Integer | `0` |
| varActiveYear | Integer | `0` |
| varETag | String | `''` |
| varRegistryItemId | Integer | `0` |
| varNextCounter | Integer | `0` |

Order matters — `varCorrelationId` must be initialized before any LogEvent calls.

---

## Step 4 — Log Started

Add **Run a Child Flow** action:
- Child flow: `CF_DocCentralV3_LogEvent`
- Inputs:

| Parameter | Value |
|---|---|
| eventType | `GenerateRegistryNumber` |
| eventCategory | `RegistryNumber` |
| severity | `Info` |
| status | `Started` |
| correlationId | `variables('varCorrelationId')` |
| flowName | `CF_DocCentralV3_GenerateRegistryNumber` |
| flowRunId | `workflow().run.name` |
| source | `CloudFlow` |
| userEmail | `triggerBody()?['text_initiatorEmail']` |
| message | `Pokretanje generisanja delovodnog broja.` |

---

## Step 5 — Try scope

Add a **Scope** action. Name it `Try`. All main logic goes inside.

### 5a — Read App Config: get active registry book item

Add **SharePoint — Get items** inside Try:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter Query: UNKNOWN — use the App Config key that identifies the active registry book row. See `appconfig-field-mapping.md` for the key names once AppConfig.csv is available.
- Top Count: `1`

Add **Condition**: check that the Get items result is not empty (`length(body('Get_items')?['value'])` equals `0`).
- Yes (empty): Log failure with error code `REGISTRY_BOOK_NOT_FOUND`, set success=false, compose failure response, terminate flow via Respond action.
- No: continue.

### 5b — Extract App Config values

Add **Compose** actions to extract values from the first App Config item:

```
varMaxRetries — set to: int(first(body('Get_items')?['value'])?['UNKNOWN_MaxRetriesColumn'])
varActiveYear — set to: int(first(body('Get_items')?['value'])?['UNKNOWN_ActiveYearColumn'])
varCounterValue — set to: int(first(body('Get_items')?['value'])?['UNKNOWN_CounterColumn'])
varRegistryItemId — set to: int(first(body('Get_items')?['value'])?['ID'])
varETag — set to: first(body('Get_items')?['value'])?['@odata.etag']
```

Use **Set variable** actions rather than Compose if the values are needed later in the loop.

Replace `UNKNOWN_*Column` with actual internal column names from App Config. See `appconfig-field-mapping.md`.

### 5c — Do-Until loop

Add **Do Until** action:
- Exit condition: `or(equals(variables('varSuccess'), true), greaterOrEquals(variables('varRetryCount'), variables('varMaxRetries')))`
- Limit Count: `15` (safety ceiling — prevents infinite loop if exit condition logic has a bug)
- Limit Timeout: `PT2M` (2 minutes total loop timeout)

Inside the Do-Until:

**5c-i — Calculate next counter**

Add **Set variable** — varNextCounter:
```
add(variables('varCounterValue'), 1)
```

**5c-ii — Format DelovodniBroj**

Add **Set variable** — varDelovodniBroj.
The format string comes from App Config (UNKNOWN key). Typical example:
```
concat(string(variables('varNextCounter')), '/', string(variables('varActiveYear')))
```
Replace with the actual format expression derived from the App Config format string value.

**5c-iii — PATCH counter with If-Match (SharePoint HTTP Request)**

Add **SharePoint — Send an HTTP Request** action:
- Connection: CR_DocCentralV3_SharePoint
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Method: `PATCH`
- Uri: `_api/web/lists/getbytitle('@{parameters(''gpdoccen_EV_DocCentralV3_lstAppConfig'')}'/items(@{variables('varRegistryItemId')})`
- Headers:
  ```
  Accept: application/json;odata=nometadata
  Content-Type: application/json;odata=nometadata
  IF-MATCH: @{variables('varETag')}
  X-HTTP-Method: MERGE
  ```
- Body:
  ```json
  {
    "UNKNOWN_CounterColumn": @{variables('varNextCounter')}
  }
  ```
  Replace `UNKNOWN_CounterColumn` with the actual internal column name.

**5c-iv — Condition on HTTP status code**

Add **Condition**:
- Left: `outputs('Send_an_HTTP_request_to_SharePoint')?['statusCode']`
- Operator: `is equal to`
- Right: `200`

**Yes branch (200 — lock acquired):**

1. Set varSuccess = `true`
2. Set varCounterValue = varNextCounter (so the response is correct)

**No branch — nested Condition:**
- Left: `outputs('Send_an_HTTP_request_to_SharePoint')?['statusCode']`
- Operator: `is equal to`
- Right: `412`

**Yes (412 — conflict, retry):**

1. Set varRetryCount = `add(variables('varRetryCount'), 1)`
2. Log Retried via CF_DocCentralV3_LogEvent:
   - eventType: `GenerateRegistryNumber`
   - severity: `Warning`
   - status: `Retried`
   - message: `concat('ETag conflict na pokušaju ', string(variables('varRetryCount')), '. Ponovni pokušaj.')`
3. Add **Delay** action: count = expression `rand(500, 1500)`, unit = Millisecond. (Note: Power Automate Delay minimum is 1 second; use `rand(1,2)` seconds if millisecond delay is not available — see `power-automate-build-notes.md`.)
4. Re-read App Config item to get fresh ETag and current counter:
   - Add **SharePoint — Get item** (single item by ID):
     - List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
     - ID: `variables('varRegistryItemId')`
   - Update varETag: `outputs('Get_item')?['body']?['@odata.etag']`
   - Update varCounterValue: `int(outputs('Get_item')?['body']?['UNKNOWN_CounterColumn'])`

**No (other error — not 200 or 412):**

Throw / terminate to Catch scope by adding a **Terminate** action with Status = Failed, Code = `REGISTRY_NUMBER_GENERATION_FAILED`, Message = `concat('HTTP PATCH failed with status ', string(outputs('Send_an_HTTP_request_to_SharePoint')?['statusCode']))`.

---

### 5d — Check max retries exhausted

After the Do-Until, add **Condition**:
- Left: `variables('varSuccess')`
- Operator: `is equal to`
- Right: `false`

If `varSuccess = false` after the loop, max retries were hit. Add Log call (Error) + compose failure response + Respond action.

---

### 5e — Uniqueness safety check

After the Do-Until (only reached if varSuccess = true), add **SharePoint — Get items**:
- List Name: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- Filter Query: `DelovodniBroj eq '@{variables('varDelovodniBroj')}'`
- Top Count: `1`

Add **Condition**: `length(body('Get_items_SviPredmeti')?['value'])` greater than `0`.

If duplicate found:
1. Log Critical:
   - eventType: `SystemError`
   - severity: `Critical`
   - message: `concat('Duplikat delovodnog broja detektovan: ', variables('varDelovodniBroj'))`
   - errorCode: `REGISTRY_NUMBER_DUPLICATE`
2. Compose failure response.
3. Respond to caller with failure.

---

## Step 6 — Catch scope

Add a second **Scope** action after Try. Name it `Catch`.

Run after settings (click the three dots → Configure run after):
- Uncheck `is successful`
- Check `has failed`
- Check `has timed out`

Inside Catch:

1. Log Failed via CF_DocCentralV3_LogEvent:
   - eventType: `PowerAutomateError`
   - severity: `Error`
   - status: `Failed`
   - correlationId: `variables('varCorrelationId')`
   - message: `Greška u generisanju delovodnog broja.`
   - errorCode: `REGISTRY_NUMBER_GENERATION_FAILED`
   - errorMessage: `result('Try')?[0]?['error']?['message']`
2. Compose failure response object.
3. Add **Respond to a PowerApp or flow** — failure outputs (see Step 8).

---

## Step 7 — Log Success and respond

After the Try and Catch scopes, add a **Condition**:
- Left: `result('Try')?[0]?['status']`
- Operator: `is equal to`
- Right: `Succeeded`

**Yes branch:**

1. Log Success via CF_DocCentralV3_LogEvent:
   - eventType: `GenerateRegistryNumber`
   - severity: `Info`
   - status: `Success`
   - message: `concat('Delovodni broj generisan: ', variables('varDelovodniBroj'))`
2. Add **Respond to a PowerApp or flow** — success outputs.

**No branch:** Already handled by Catch scope. Add a **Respond** action here with failure response as a safety net.

---

## Step 8 — Respond to a PowerApp or flow

Add **Respond to a PowerApp or flow** action in both branches.

**Success outputs:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `true` |
| delovodniBroj | Text | `variables('varDelovodniBroj')` |
| counterValue | Integer | `variables('varCounterValue')` |
| activeYear | Integer | `variables('varActiveYear')` |
| correlationId | Text | `variables('varCorrelationId')` |

**Failure outputs:**

| Output name | Type | Value |
|---|---|---|
| success | Boolean | `false` |
| delovodniBroj | Text | `''` |
| counterValue | Integer | `0` |
| activeYear | Integer | `0` |
| correlationId | Text | `variables('varCorrelationId')` |
| message | Text | `'Registry number generation failed.'` |
| errorCode | Text | `'REGISTRY_NUMBER_GENERATION_FAILED'` |

---

## Step 9 — Save and verify

1. Save the flow.
2. Verify it appears inside the DocCentralV3 solution.
3. Run test cases TC-001 through TC-004 before integrating with CF_DocCentralV3_CreateDocument.
4. Add description: `"v1.0 — initial implementation. CF_DocCentralV3_GenerateRegistryNumber. DocCentralV3 solution."`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_GenerateRegistryNumber`
- [ ] Trigger is Power Apps (V2) with two inputs: initiatorEmail (Text), correlationId (Text)
- [ ] All 10 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Log Started called before Try scope
- [ ] App Config read inside Try — empty result handled (REGISTRY_BOOK_NOT_FOUND)
- [ ] Do-Until exits on varSuccess=true OR varRetryCount >= varMaxRetries
- [ ] PATCH uses SharePoint Send an HTTP Request (not standard connector)
- [ ] If-Match header set to varETag
- [ ] 200 → varSuccess=true
- [ ] 412 → increment retry, log Retried (Warning), delay, re-read ETag
- [ ] Other status → Terminate (triggers Catch)
- [ ] Max retries exhausted handled after Do-Until
- [ ] Uniqueness safety check against Svi predmeti after success
- [ ] REGISTRY_NUMBER_DUPLICATE triggers Critical log + failure
- [ ] Catch scope runs on failed/timed out only
- [ ] Catch never calls this flow (no infinite loop)
- [ ] Success and failure respond via Respond to a PowerApp or flow
- [ ] All SharePoint actions use CR_DocCentralV3_SharePoint
- [ ] All EV references use @parameters() syntax
- [ ] Flow is inside DocCentralV3 solution
