# Field mapping: CF_DocCentralV3_ArchiveDocument

## Svi predmeti — fields read

| Display name | Internal name | Expression | Notes |
|---|---|---|---|
| (ID) | system | `outputs('Get_SviPredmeti_Item_ForArchive')?['body']?['ID']` | Not read explicitly — ID passed as input |
| Stanje | UNKNOWN | `outputs('Get_SviPredmeti_Item_ForArchive')?['body']?['UNKNOWN_Stanje']` | Pre-condition check: must = `Zavedeno` |
| @odata.etag | system | `outputs('Get_SviPredmeti_Item_ForArchive')?['body']?['@odata.etag']` | Used in IF-MATCH header |

---

## Svi predmeti — fields written (HTTP PATCH)

All internal names UNKNOWN — resolve before building.

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| Stanje | UNKNOWN | `'Arhivirano'` | Always set to Arhivirano |
| ArhivskiZnak | UNKNOWN | `item()?['arhivskiZnak']` from trigger input | Archive classification code |
| Arhivirano | UNKNOWN | `utcNow()` | **Display name confirmed.** Date of archiving. |
| Arhivirao | UNKNOWN | `triggerBody()?['text_initiatorEmail']` | **Display name confirmed.** Who archived. |

---

## PATCH body structure

```json
{
  "__metadata": { "type": "SP.Data.SviPredmetiListItem" },
  "UNKNOWN_Stanje": "Arhivirano",
  "UNKNOWN_ArhivskiZnak": "<item().arhivskiZnak>",
  "UNKNOWN_Arhivirano": "<utcNow()>",
  "UNKNOWN_Arhivirao": "<triggerBody()['text_initiatorEmail']>"
}
```

Replace all `UNKNOWN_*` with actual internal column names before building.
The `__metadata type` value may differ from `SP.Data.SviPredmetiListItem` — confirm exact type from the list schema.

---

## PATCH HTTP headers

| Header | Value |
|---|---|
| Accept | `application/json;odata=verbose` |
| Content-Type | `application/json;odata=verbose` |
| IF-MATCH | `variables('varItemETag')` |
| X-HTTP-Method | `MERGE` |
| X-RequestDigest | `variables('varRequestDigest')` |

---

## PATCH URI

```
concat(
  "@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')",
  "/_api/web/lists/GetByTitle('",
  "@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')",
  "')/items(",
  "<documentItemId>",
  ")"
)
```

---

## App Config — fields read

| Purpose | Filter | Notes |
|---|---|---|
| Valid archive signs | UNKNOWN category key | Returns array of allowed sign values; UNKNOWN internal column name for sign value |

Expression to extract sign values from App Config results (UNKNOWN internal column name):
```
map(body('Get_ValidArchiveSigns')?['value'], item()?['UNKNOWN_SignValue'])
```

---

## Request digest

| Action | Expression |
|---|---|
| POST `_api/contextinfo` | `body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']` |

---

## Failure entry structure (varFailuresJson)

Each failure appended during the loop is a JSON object:
```json
{
  "documentItemId": <integer>,
  "delovodniBroj": "<string>",
  "errorCode": "<string>",
  "message": "<string>"
}
```

Accumulated with:
```
string(concat(json(variables('varFailuresJson')), array(outputs('Build_FailureEntry_*'))))
```

---

## 412 — idempotency check

After a 412 on PATCH, re-read the item and check its Stanje:

| Re-read Stanje | Action |
|---|---|
| `Arhivirano` | Treat as success — increment archivedCount, log success, continue |
| Anything else | Append CONCURRENT_UPDATE failure, increment failedCount, continue |

---

## Open items

| Item | Status |
|---|---|
| Svi predmeti Stanje internal column name | UNKNOWN |
| Svi predmeti ArhivskiZnak internal column name | UNKNOWN |
| Svi predmeti Arhivirano internal column name | UNKNOWN — display name confirmed |
| Svi predmeti Arhivirao internal column name | UNKNOWN — display name confirmed |
| `__metadata type` for Svi predmeti | UNKNOWN — check with `$select=__metadata` GET |
| App Config key for archive signs category | UNKNOWN |
| App Config internal column name for sign value | UNKNOWN |
| Maximum documents per call | UNKNOWN — suggest 50 |
