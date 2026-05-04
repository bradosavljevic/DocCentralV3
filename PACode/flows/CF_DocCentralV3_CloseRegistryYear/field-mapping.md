# Field mapping: CF_DocCentralV3_CloseRegistryYear

## App Config — fields read (active registry book item)

| Display name | Internal name | Expression | Notes |
|---|---|---|---|
| (ID) | system | `first(body('Get_ActiveRegistryBook')?['value'])?['ID']` | Used as PATCH target item ID |
| Active year | UNKNOWN | `first(body('Get_ActiveRegistryBook')?['value'])?['UNKNOWN_ActiveYear']` | Compared against input yearToClose |
| IsClosed | UNKNOWN | `outputs('Get_RegistryBookItem')?['UNKNOWN_IsClosed']` | Precondition 2 check |
| FormatString | UNKNOWN | `outputs('Get_RegistryBookItem')?['UNKNOWN_FormatString']` | Copied to new registry book |
| @odata.etag | system | `first(body('Get_ActiveRegistryBook')?['value'])?['@odata.etag']` | Used in IF-MATCH header |

**Get items filter for active registry book (UNKNOWN):**
The filter must return the current open registry book. Depending on App Config schema:
- Option A: Filter by `IsClosed eq false` — returns the one open book.
- Option B: Filter by a category key column (UNKNOWN) where value = `DelovodnjaKnjiga`.
- Option C: Both conditions combined.

Top: 1. Order by year descending (if multiple books exist with IsClosed = false, take the latest).

---

## App Config — fields written: close existing year (PATCH)

All internal names UNKNOWN — resolve before building.

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| IsClosed | UNKNOWN | `true` | Marks year as closed |
| ClosedAt | UNKNOWN | `utcNow()` | Timestamp of closing |
| ClosedBy | UNKNOWN | `triggerBody()?['text_initiatorEmail']` | Who closed the year |

---

## App Config — fields written: create new registry book (CREATE)

All internal names UNKNOWN — resolve before building. All values either computed or copied from closing year.

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| Category / Type | UNKNOWN | UNKNOWN value | Same as existing registry book items |
| ActiveYear | UNKNOWN | `variables('varNewYear')` | closedYear + 1 |
| Counter | UNKNOWN | `0` | UNKNOWN — confirm with GenerateRegistryNumber pattern |
| IsClosed | UNKNOWN | `false` | New book starts open |
| FormatString | UNKNOWN | `outputs('Get_RegistryBookItem')?['UNKNOWN_FormatString']` | Copied from closing year |

---

## PATCH body structure — close year

```json
{
  "__metadata": { "type": "SP.Data.AppConfigListItem" },
  "UNKNOWN_IsClosed": true,
  "UNKNOWN_ClosedAt": "@{utcNow()}",
  "UNKNOWN_ClosedBy": "@{triggerBody()?['text_initiatorEmail']}"
}
```

Replace all `UNKNOWN_*` with actual internal column names before building.
The `__metadata type` value may differ — check with `$select=__metadata` GET against App Config list.

---

## PATCH HTTP headers

| Header | Value |
|---|---|
| Accept | `application/json;odata=verbose` |
| Content-Type | `application/json;odata=verbose` |
| IF-MATCH | `variables('varRegistryBookETag')` |
| X-HTTP-Method | `MERGE` |
| X-RequestDigest | `variables('varRequestDigest')` |

---

## PATCH URI — App Config close year

```
concat(
  "@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')",
  "/_api/web/lists/GetByTitle('",
  "@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')",
  "')/items(",
  variables('varRegistryBookItemId'),
  ")"
)
```

---

## Svi predmeti — fields read for precondition 3

| Purpose | Filter | Notes |
|---|---|---|
| Check for non-archived documents | `UNKNOWN_Year eq <varYearToClose> and UNKNOWN_Stanje ne 'Arhivirano'` | Top 1 — existence check only |

UNKNOWN: Svi predmeti year column name. This may be a computed year extracted from the filing date,
or a dedicated year column. To be confirmed from Svi predmeti schema.

Alternative filter if no dedicated year column (using date range — UNKNOWN column name for FilingDate):
```
UNKNOWN_FilingDate ge '2026-01-01T00:00:00Z' and UNKNOWN_FilingDate lt '2027-01-01T00:00:00Z' and UNKNOWN_Stanje ne 'Arhivirano'
```

---

## RezervisaniBrojevi — fields read for precondition 4

| Purpose | Filter | Notes |
|---|---|---|
| Check for remaining reserved numbers | `UNKNOWN_Year eq <varYearToClose>` | Top 1 — existence check only |

UNKNOWN: RezervisaniBrojevi year column name.

---

## Request digest

| Action | Expression |
|---|---|
| POST `_api/contextinfo` | `body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']` |

---

## Active year pointer (conditional)

If App Config contains a separate item with Key = `ActiveYear` (or similar), it must also be patched after creating the new registry book.

UNKNOWN — depends on App Config schema. Check during App Config audit.

---

## Open items

| Item | Status |
|---|---|
| App Config filter for active registry book | UNKNOWN — IsClosed = false, or by category key |
| App Config ActiveYear internal column name | UNKNOWN |
| App Config IsClosed internal column name | UNKNOWN |
| App Config ClosedAt internal column name | UNKNOWN |
| App Config ClosedBy internal column name | UNKNOWN |
| App Config FormatString internal column name | UNKNOWN |
| App Config Counter initial value (0 or 1) | UNKNOWN — align with GenerateRegistryNumber |
| App Config `__metadata type` for Create | UNKNOWN |
| Whether separate active year pointer item exists | UNKNOWN |
| Svi predmeti year filter column name | UNKNOWN |
| Svi predmeti Stanje internal column name | UNKNOWN |
| RezervisaniBrojevi year column name | UNKNOWN |
