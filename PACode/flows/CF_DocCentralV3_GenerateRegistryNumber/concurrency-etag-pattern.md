# ETag / If-Match concurrency pattern: CF_DocCentralV3_GenerateRegistryNumber

## Problem

Multiple users submitting documents simultaneously will each call
CF_DocCentralV3_GenerateRegistryNumber at the same moment.
Without concurrency control, two flows could both read counter = 41,
both write counter = 42, and produce the same delovodni broj for two different documents.

Registry numbers must be unique. A race condition here is not acceptable.

---

## Solution: Optimistic locking via SharePoint ETag

SharePoint assigns a version identifier (ETag) to every list item. The ETag changes
each time the item is modified. By reading the ETag and then including it in the
PATCH request as an `If-Match` header, the update is conditional:

- If the item has not changed since it was read: PATCH succeeds (HTTP 200).
- If another process modified the item first (ETag mismatch): SharePoint rejects the PATCH (HTTP 412 Precondition Failed).

The 412 response is not an error — it is the concurrency signal. The flow retries
by reading the item again (fresh ETag + updated counter), then attempts the PATCH again.

---

## Algorithm

```
READ App Config item → capture (counterValue, ETag, activeYear, formatString)
SET varRetryCount = 0

DO UNTIL varSuccess = true OR varRetryCount >= varMaxRetries:

    nextCounter = counterValue + 1
    delovodniBroj = format(nextCounter, activeYear)

    PATCH _api/.../items(registryItemId)
        body: { CounterColumn: nextCounter }
        header: If-Match: <ETag>

    IF response = 200:
        varSuccess = true          ← lock acquired, number is ours
        varCounterValue = nextCounter

    ELSE IF response = 412:
        varRetryCount++
        Log Retried (Warning)
        Delay rand(500–1500) ms    ← jitter to prevent thundering herd
        RE-READ item → refresh (counterValue, ETag)

    ELSE:
        Terminate (unexpected HTTP error) → Catch scope

IF NOT varSuccess:
    Log Failed (Error) → return REGISTRY_NUMBER_GENERATION_FAILED

SAFETY CHECK: query Svi predmeti for delovodniBroj
IF found: Log Critical → return REGISTRY_NUMBER_DUPLICATE

Log Success (Info)
RETURN { success: true, delovodniBroj, counterValue, activeYear }
```

---

## Why optimistic locking (not a SharePoint lock or queue)

| Option | Problem |
|---|---|
| Read-then-write without locking | Race condition — duplicate numbers guaranteed under concurrency |
| SharePoint item lock (check-out/in) | Only available in document libraries, not custom lists |
| Pessimistic lock (custom flag column) | Complex; flag can be orphaned if flow crashes, causing deadlock |
| Azure Service Bus / queue | Requires premium connector or additional infrastructure |
| ETag / If-Match | Native SharePoint feature; no extra infrastructure; atomic at the HTTP layer |

ETag is the correct choice for this scenario. SharePoint guarantees that two simultaneous PATCH requests with the same ETag will result in exactly one 200 and one 412.

---

## Random jitter rationale

Without jitter, two flows hitting a 412 simultaneously would both retry at the same instant,
collide again, and repeat — a classic thundering herd.

Using `rand(500, 1500)` milliseconds of delay means the two retrying flows are
unlikely to retry at exactly the same time.

**Power Automate note:** The Delay action's minimum unit is seconds, not milliseconds.
Use `rand(1, 2)` seconds if millisecond precision is not available.
For the purposes of this design, the jitter range is expressed in ms to convey the intent;
adapt to the actual Delay action capability. See `power-automate-build-notes.md`.

---

## ETag format in SharePoint REST API

SharePoint ETags returned from the REST API look like:

```
"\"3f2504e0-4f89-11d3-9a0c-0305e82c3301,42\""
```

The outer double-quotes are part of the JSON string; the inner escaped quotes and comma-separated
version are part of the ETag value. When passing to `If-Match`, use the full value as returned —
do not strip the quotes.

In Power Automate expressions, the ETag is available as:
```
first(body('Get_items')?['value'])?['@odata.etag']
```
or for single-item GET:
```
outputs('Get_item')?['body']?['@odata.etag']
```

Use this value directly in the `IF-MATCH` header field.

---

## Retry limit

Default max retries: **5**.

Under normal conditions, an ETag conflict resolves in 1–2 retries (rare simultaneous submissions).
Five retries allows for burst scenarios (e.g. bulk document import) while bounding total wait time.

Maximum wall-clock time in worst case:
- 5 retries × 1.5 seconds jitter max = 7.5 seconds
- Plus 5 × SharePoint HTTP roundtrip ≈ 5–10 seconds
- Total worst case ≈ 15–17 seconds

Power Automate default action timeout is 2 minutes — well within this bound.

If 5 retries are exhausted, the flow returns `REGISTRY_NUMBER_GENERATION_FAILED`.
The caller (CF_DocCentralV3_CreateDocument) stops and informs the Canvas App.
The user must retry document submission manually.

Max retries is configurable via App Config (UNKNOWN key). See `appconfig-field-mapping.md`.

---

## Uniqueness safety check

After a successful PATCH, the flow performs an additional safety check by querying
Svi predmeti for any existing item with the same delovodniBroj.

This check is not a substitute for ETag locking — it is a defence-in-depth guard
against unlikely scenarios such as:
- A legacy flow or manual SharePoint edit that wrote the same number.
- A bug in the format string that produces collisions across years.
- Data migration artefacts.

If a duplicate is found, the flow:
1. Logs a Critical severity event (errorCode: REGISTRY_NUMBER_DUPLICATE).
2. Returns failure — does NOT increment again or try a new number.
3. Leaves the App Config counter at the incremented value (the number is "used" to prevent re-use).

An administrator must investigate and manually resolve any REGISTRY_NUMBER_DUPLICATE incident.

---

## PATCH action — SharePoint Send an HTTP Request

The standard SharePoint connector's "Update item" action does not support custom HTTP headers.
The `If-Match` header cannot be set using the standard connector.

Use the **SharePoint — Send an HTTP Request** action instead.

Key fields:

| Field | Value |
|---|---|
| Method | PATCH |
| Uri | `_api/web/lists/getbytitle('<ListName>')/items(<ID>)` |
| IF-MATCH header | ETag value from previous GET |
| X-HTTP-Method header | MERGE |
| Content-Type header | `application/json;odata=nometadata` |
| Accept header | `application/json;odata=nometadata` |
| Body | `{"CounterColumn": nextCounterValue}` |

The Uri list name must reference the environment variable — see `power-automate-build-notes.md`
for how to embed EV references inside the URI string of an HTTP Request action.

---

## What callers must know

CF_DocCentralV3_CreateDocument calls this flow and must:
- Pass correlationId to link all retried attempts in the audit log under one correlation chain.
- Never retry this flow externally — internal retry handles it.
- Treat `success = false` as a terminal failure for the document creation operation.
- Not attempt to re-use a delovodniBroj if the flow returns failure — do not guess or re-call.
