# Power Automate build notes: CF_DocCentralV3_GenerateRegistryNumber

## Purpose of this document

Practical notes and gotchas for building this flow in the Power Automate maker portal.
Read this alongside `implementation-steps.md` and `concurrency-etag-pattern.md` before starting.

---

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_GenerateRegistryNumber |
| Solution | DocCentralV3 |
| Trigger | Power Apps (V2) — child flow pattern |
| Called by | CF_DocCentralV3_CreateDocument |
| Connection reference | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAuditLog |

---

## Child flow trigger — NOT a standalone flow

This flow uses **Power Apps (V2)** as its trigger to enable the child flow pattern.
It has only 2 trigger inputs, both Text type:

| Input name | Type | Internal reference |
|---|---|---|
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

The flow is not designed to be run from the Power Automate test button with manual input —
test by calling it from a temporary test flow using "Run a Child Flow".

---

## Send an HTTP Request to SharePoint — critical note

The standard **SharePoint — Update item** connector action does not support custom HTTP headers.
The `If-Match` header required for ETag concurrency control cannot be set via the standard connector.

You must use the **SharePoint — Send an HTTP Request** action for the PATCH call.

Find this action by searching "Send an HTTP request to SharePoint" in the action search.
It is a standard (non-premium) action in the SharePoint connector.

### PATCH request configuration

| Field | Value |
|---|---|
| Site Address | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| Method | POST (set X-HTTP-Method: MERGE — see headers) |
| Uri | `_api/web/lists/getbytitle('@{parameters(''gpdoccen_EV_DocCentralV3_lstAppConfig'')}'/items(@{variables('varRegistryItemId')})` |

**Headers** (add as key-value pairs):
```
Accept               application/json;odata=nometadata
Content-Type         application/json;odata=nometadata
IF-MATCH             @{variables('varETag')}
X-HTTP-Method        MERGE
```

**Body** (raw JSON):
```json
{
  "UNKNOWN_CounterColumn": @{variables('varNextCounter')}
}
```

Replace `UNKNOWN_CounterColumn` with the actual internal column name from App Config.

**Important:** The Method field in the Send an HTTP Request action must be set to `POST`,
not `PATCH`. The `X-HTTP-Method: MERGE` header tells SharePoint to treat it as a PATCH.
This is the SharePoint REST API convention for list item updates.

---

## EV reference inside the HTTP Request URI

Environment variables cannot be embedded in the URI field of the HTTP Request action
using the standard `@parameters()` syntax directly — the URI field may not accept it
as an expression in the same way as connector action fields.

Use string interpolation via an expression:

```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_lstAppConfig'),
  ''')/items(',
  string(variables('varRegistryItemId')),
  ')'
)
```

Switch the URI field to Expression mode and paste the concat expression.
Single quotes inside the list name must be doubled: `'listname'` → `''listname''` inside concat.

---

## Reading the ETag from Get items response

After **SharePoint — Get items**, the ETag for the first result is at:
```
first(body('Get_items')?['value'])?['@odata.etag']
```

After **SharePoint — Get item** (single item by ID), the ETag is at:
```
outputs('Get_item')?['body']?['@odata.etag']
```

Use **Set variable** — varETag with this expression. The ETag includes surrounding quotes
and version number, e.g. `"\"abc123,5\""`. Use it verbatim in the IF-MATCH header — do not strip quotes.

---

## Reading the HTTP response status code

After the Send an HTTP Request action, check the status code:
```
outputs('Send_an_HTTP_request_to_SharePoint')?['statusCode']
```

If you rename the action, replace `Send_an_HTTP_request_to_SharePoint` with the
action's internal name (visible in the expression editor dynamic content panel).

Expected codes:
- `200` — PATCH accepted, counter incremented.
- `412` — ETag mismatch, retry.
- Any other code — unexpected error, terminate to Catch scope.

---

## Do-Until loop configuration

| Setting | Value |
|---|---|
| Exit condition | `or(equals(variables('varSuccess'), true), greaterOrEquals(variables('varRetryCount'), variables('varMaxRetries')))` |
| Limit Count | `15` |
| Limit Timeout | `PT2M` |

The Limit Count and Timeout are safety ceilings. Under normal operation, the loop exits
via the condition expression. Do not rely on the ceiling — always test that the condition
exits correctly.

**Note:** Inside a Do-Until loop, the standard Condition action works normally.
However, Switch actions and some nested scopes may not be available inside Do-Until — use Condition actions only.

---

## Delay action — milliseconds vs seconds

Power Automate's **Delay** action minimum unit is **1 second**, not milliseconds.

The design specifies `rand(500, 1500)` ms jitter conceptually. In practice:
- Use `rand(1, 2)` seconds, or
- Use `rand(1, 3)` seconds for higher spread.

The exact range is less critical than ensuring it is non-zero and randomized.
Even a 1-second fixed delay is acceptable for v1.

---

## Environment variable references

All SharePoint actions must reference EVs via the @parameters syntax:

| EV display name | Expression |
|---|---|
| EV_DocCentralV3_SharePointSite | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| EV_DocCentralV3_lstAppConfig | `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')` |
| EV_DocCentralV3_lstSviPredmeti | `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')` |
| EV_DocCentralV3_lstAuditLog | `@parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')` |

If the logical name prefix differs (e.g. no `gpdoccen_` prefix), verify the exact logical name
in the solution's Environment Variables section before use.

---

## Try / Catch scope — run after settings

- Try scope: default (runs after the previous action succeeds).
- Catch scope: click "..." → Configure run after:
  - Uncheck `is successful`
  - Check `has failed`
  - Check `has timed out`

After both scopes, add a Condition on `result('Try')?[0]?['status']` equals `Succeeded`
to route to the success Respond action or failure Respond action.

The Catch scope contains its own Respond action as an additional safety net.

---

## CF_DocCentralV3_LogEvent calls in this flow

This flow calls CF_DocCentralV3_LogEvent four times. Each uses "Run a Child Flow" action.

| When | eventType | severity | status |
|---|---|---|---|
| Before Try | GenerateRegistryNumber | Info | Started |
| On 412 retry (inside Do-Until) | GenerateRegistryNumber | Warning | Retried |
| After Do-Until success | GenerateRegistryNumber | Info | Success |
| In Catch scope | PowerAutomateError | Error | Failed |

**Critical:** The Catch scope must NEVER call CF_DocCentralV3_GenerateRegistryNumber.
The Catch scope calls CF_DocCentralV3_LogEvent only. Calling this flow from its own Catch
would create an infinite loop.

---

## Naming actions to avoid expression conflicts

When multiple "Get items" actions exist in the flow (one for App Config, one for Svi predmeti),
rename each action clearly:

- `Get_AppConfig_RegistryBook`
- `Get_SviPredmeti_DuplicateCheck`

Rename the HTTP Request action:
- `PATCH_AppConfig_Counter`

This avoids expression name collisions and makes the flow history readable.

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_GenerateRegistryNumber`
- [ ] Trigger is Power Apps (V2) with 2 Text inputs: initiatorEmail, correlationId
- [ ] All 10 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Log Started called with correct parameters
- [ ] App Config Get items uses EV expression, not hardcoded list name
- [ ] Empty App Config result handled: REGISTRY_BOOK_NOT_FOUND error returned
- [ ] varETag set from @odata.etag field of App Config item
- [ ] Do-Until exit condition uses varSuccess and varRetryCount
- [ ] Do-Until Limit Count = 15, Limit Timeout = PT2M
- [ ] PATCH uses Send an HTTP Request (NOT standard Update item connector)
- [ ] PATCH method = POST with X-HTTP-Method: MERGE header
- [ ] IF-MATCH header set to varETag expression
- [ ] 200 response → varSuccess = true
- [ ] 412 response → retry count increment, Retried log, Delay, re-read App Config
- [ ] Other status → Terminate action (triggers Catch)
- [ ] After Do-Until: exhausted retries (varSuccess=false) handled with failure response
- [ ] Uniqueness check: Get items from Svi predmeti filtered by DelovodniBroj
- [ ] Duplicate found → REGISTRY_NUMBER_DUPLICATE Critical log + failure response
- [ ] Catch scope runs on failed/timed out only
- [ ] Catch scope calls LogEvent (PowerAutomateError/Error/Failed)
- [ ] Catch scope does NOT call CF_DocCentralV3_GenerateRegistryNumber
- [ ] Success Respond outputs: success, delovodniBroj, counterValue, activeYear, correlationId, message, errorCode
- [ ] Failure Respond outputs match the same 7 output names with failure values
- [ ] All SharePoint actions use CR_DocCentralV3_SharePoint connection reference
- [ ] Flow is inside DocCentralV3 solution
- [ ] TC-001 through TC-004 pass before integration with CF_DocCentralV3_CreateDocument
