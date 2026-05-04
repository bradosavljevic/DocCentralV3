# Power Automate build notes: CF_DocCentralV3_CloseRegistryYear

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_CloseRegistryYear |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstRezervisaniBrojevi, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog |

---

## yearToClose input — Number type and decimal_ prefix

`yearToClose` is passed as Number type from Canvas App.
Power Apps V2 trigger encodes Number inputs with the `decimal_` prefix:

```
int(triggerBody()?['decimal_yearToClose'])
```

Store as `varYearToClose` (Integer). Use the variable throughout — do not reference the trigger body again.

---

## Reading active registry book from App Config

Use **SharePoint — Get items** (standard connector action — not HTTP Request).
This is a simple list read; no REST URL construction needed.

Key design decisions:
- **Top: 1** — only one active (open) registry book should exist at any time.
- **Filter**: UNKNOWN — filter for IsClosed = false, or by category key.
- The standard connector returns `@odata.etag` on each item in the `value` array.

Access the first item:
```
first(body('Get_ActiveRegistryBook')?['value'])
```

If `body('Get_ActiveRegistryBook')?['value']` is null or empty:
```
equals(length(body('Get_ActiveRegistryBook')?['value']), 0)
```
→ return YEAR_NOT_ACTIVE (no active registry book found).

---

## ETag from Get items connector

The standard SharePoint Get items connector includes `@odata.etag` on each item in the response:
```
first(body('Get_ActiveRegistryBook')?['value'])?['@odata.etag']
```

If this is null (can happen in some SP configurations), follow up with:
```
[Get_RegistryBookItem_ETag]
SharePoint — Get item
ID: variables('varRegistryBookItemId')
```
Then read `outputs('Get_RegistryBookItem_ETag')?['body']?['@odata.etag']`.

---

## Storing the registry book item for field access

After reading with Get items, store the full item object in a Compose for easier reference:

```
[Compose: Get_RegistryBookItem]
first(body('Get_ActiveRegistryBook')?['value'])
```

Then reference fields as:
```
outputs('Get_RegistryBookItem')?['UNKNOWN_IsClosed']
outputs('Get_RegistryBookItem')?['UNKNOWN_FormatString']
outputs('Get_RegistryBookItem')?['UNKNOWN_ActiveYear']
```

---

## Precondition checks: Top 1 filter pattern

Preconditions 3 and 4 use existence checks — not full counts.
`Top: 1` is sufficient and avoids loading large datasets.

Check for any result:
```
greater(length(body('Get_NonArchivedDocs')?['value']), 0)
```

This returns true if at least one non-archived document exists for this year.

---

## Terminate Succeeded for precondition failures

All precondition failures (YEAR_NOT_ACTIVE, YEAR_ALREADY_CLOSED, DOCUMENTS_NOT_ALL_ARCHIVED,
RESERVED_NUMBERS_EXIST) must use **Terminate Succeeded** — not Terminate Failed.

Reason: Terminate Failed triggers the Catch scope. Catch scope attempts to Respond with a failure
response. But the precondition branches already called Respond. A second Respond causes a flow error.

See `PACode/flows/CF_DocCentralV3_UseReservedNumber/power-automate-build-notes.md` for the full explanation.

---

## PATCH App Config — HTTP Request (not standard connector)

The standard SharePoint Update item connector does not support the `IF-MATCH` header.
Use **SharePoint — Send an HTTP Request** with method POST and `X-HTTP-Method: MERGE`.

```
Method: POST (not PATCH — Power Automate uses POST + X-HTTP-Method: MERGE)
Uri: _api/web/lists/GetByTitle('<lstAppConfig>')/items(<varRegistryBookItemId>)
Headers:
  IF-MATCH: <varRegistryBookETag>
  X-HTTP-Method: MERGE
  X-RequestDigest: <varRequestDigest>
  Content-Type: application/json;odata=verbose
  Accept: application/json;odata=verbose
```

The `IF-MATCH` header expects the raw ETag value from `@odata.etag` — which includes surrounding quotes:
`"abc123"` (with quotes). Pass it as-is. SharePoint requires the quotes to be present.

---

## Request digest

Obtain once, before the PATCH:
```
[Send an HTTP Request: Get_RequestDigest]
Method: POST
Uri: _api/contextinfo
Headers: Accept: application/json;odata=verbose

[Compose: Extract_RequestDigest]
body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']
```

The digest is used only once (one PATCH). No need to refresh.

---

## Creating new registry book item — standard connector

Use **SharePoint — Create item** (standard connector) for the new registry book creation.
No If-Match needed — this is a new item, not an update.

This is the same connector action used by other flows for item creation.
Dynamic content must map to the correct internal column names (UNKNOWN — resolve before building).

---

## 412 on PATCH — no retry, immediate failure

Unlike ArchiveDocument (which re-reads for idempotency), CloseRegistryYear treats 412 as an
unrecoverable state and returns CONCURRENT_UPDATE immediately.

Reasoning: A 412 on the App Config registry book item during year closing means another admin
is simultaneously performing the same operation. This is an exceptional condition. Retrying
blindly could result in a double-close attempt. The correct action is to fail and let the user
refresh and recheck.

---

## Filter expression format for SharePoint Get items

SharePoint OData filters use single equals (`eq`), not double (`==`):
```
UNKNOWN_Year eq 2026 and UNKNOWN_Stanje ne 'Arhivirano'
```

String values are wrapped in single quotes. Integer values are not quoted.
Boolean values (for IsClosed): `UNKNOWN_IsClosed eq false` — no quotes.

If filtering by date range (year fallback):
```
UNKNOWN_FilingDate ge '2026-01-01T00:00:00Z' and UNKNOWN_FilingDate lt '2027-01-01T00:00:00Z'
```

---

## Partial-complete state: year closed, new book not created

If the PATCH (close year) succeeds but Create item (new registry book) fails:
- The year is closed. No new registry book exists.
- CreateDocument will fail (no open registry book).
- This is an admin-recoverable state: manually create the 2027 registry book in App Config.
- Log as Error / Critical. Return CLOSE_YEAR_FAILED.

No automatic rollback is implemented. The year-close PATCH cannot be reversed by a flow.

---

## EV expression in URI

Environment variables in URI strings must use the expression (not `@parameters()` literal):
```
concat(
  parameters('gpdoccen_EV_DocCentralV3_SharePointSite'),
  "/_api/web/lists/GetByTitle('",
  parameters('gpdoccen_EV_DocCentralV3_lstAppConfig'),
  "')/items(",
  variables('varRegistryBookItemId'),
  ")"
)
```

In Power Automate expression editor, use `@parameters('...')` syntax.
The single quotes around the list title are SharePoint REST syntax — they are literal, not expression delimiters.
In a string built with `concat()`, they do not need doubling. If the list name is hardcoded in a URI string (not inside concat), use `''` to escape the single quote.

---

## Action naming convention

| Purpose | Name |
|---|---|
| Get active registry book | `Get_ActiveRegistryBook` |
| Store registry book item | `Get_RegistryBookItem` |
| Check non-archived docs | `Get_NonArchivedDocs` |
| Check reserved numbers | `Get_ReservedNumbers` |
| Get request digest | `Get_RequestDigest`, `Extract_RequestDigest` |
| PATCH close year | `PATCH_AppConfig_CloseYear` |
| Create new registry book | `Create_NewRegistryBook` |
| Update active year pointer (if needed) | `PATCH_AppConfig_ActiveYear` |

---

## Checklist

- [ ] yearToClose parsed as int(decimal_yearToClose)
- [ ] Get_ActiveRegistryBook uses Top 1; empty result → YEAR_NOT_ACTIVE
- [ ] varRegistryBookETag read from Get_ActiveRegistryBook result
- [ ] Precondition 1: year match
- [ ] Precondition 2: IsClosed = false check
- [ ] Precondition 3: Svi predmeti — Top 1, filter by year AND Stanje ≠ Arhivirano
- [ ] Precondition 4: RezervisaniBrojevi — Top 1, filter by year
- [ ] All precondition failures use Terminate Succeeded
- [ ] Request digest obtained before PATCH
- [ ] PATCH uses IF-MATCH, X-HTTP-Method: MERGE
- [ ] 412: log Error, return CONCURRENT_UPDATE, Terminate Succeeded
- [ ] varNewYear = varYearToClose + 1
- [ ] New registry book created with FormatString copied, Counter = 0, IsClosed = false
- [ ] Partial-complete state logged as Error if Create fails after PATCH succeeds
- [ ] Active year pointer updated if separate App Config item (UNKNOWN)
- [ ] Log Success before Respond success
- [ ] Catch scope: log Error, respond failure
- [ ] Flow inside DocCentralV3 solution
