# App Config field mapping: CF_DocCentralV3_GenerateRegistryNumber

## Purpose

Documents which App Config SharePoint list columns this flow reads, what they contain,
and what internal column names to use in expressions.

**Status: All App Config keys and internal column names are UNKNOWN.**
AppConfig.csv has not been provided. Do not build the flow until these values are confirmed.
Update this file once AppConfig.csv is available or once the App Config list columns are confirmed
by reading them from the actual SharePoint list.

---

## How App Config is structured

App Config is a SharePoint list where each row is a configuration entry.
The flow finds the active registry book row by filtering on a key column
(typically a `Key` or `ConfigKey` single-line-of-text column) with a known value.

The exact filter column name and the key string values used to identify the registry book
row are all UNKNOWN until AppConfig.csv is provided.

---

## Registry book App Config row

The flow expects exactly one App Config row representing the active registry book.

| What the flow needs | App Config display name | Internal column name | Type | Notes |
|---|---|---|---|---|
| Filter key to find this row | UNKNOWN | UNKNOWN | Single line of text | Must return exactly 1 row when filtered |
| Active year | UNKNOWN | UNKNOWN | Number | Current calendar year for the registry (e.g. 2026) |
| Current counter | UNKNOWN | UNKNOWN | Number | Last used registry number. Flow will read this and write `counter + 1`. |
| Format string | UNKNOWN | UNKNOWN | Single line of text | Template for formatting delovodniBroj. See Format string section below. |
| Max retries | UNKNOWN | UNKNOWN | Number | Maximum ETag retry attempts. Flow default: 5 if not found in App Config. |
| SharePoint item ID | ID (system column) | ID | Integer | Used in the PATCH URI: `_api/web/lists/.../items(ID)` |
| ETag | @odata.etag (system field) | @odata.etag | String | Returned in Get items response body. Used in If-Match header for PATCH. |

---

## How to find internal column names

1. Open the App Config list in SharePoint.
2. Go to List Settings → Columns → click each column name.
3. The internal name appears in the URL parameter `Field=<InternalName>`.

Or use the SharePoint REST API to list all columns:
```
GET _api/web/lists/getbytitle('AppConfigListName')/fields?$filter=Hidden eq false&$select=Title,InternalName,TypeAsString
```

---

## Format string

The delovodniBroj is formatted using a template read from App Config.
The exact format is organisation-specific and stored in App Config rather than hardcoded.

Typical Serbian registry format: `{counter}/{year}` → `42/2026`

The flow must apply the format string to produce the final `varDelovodniBroj` value.
If the format string uses a simple pattern like `{counter}/{year}`, implement this as:
```
concat(string(variables('varNextCounter')), '/', string(variables('varActiveYear')))
```

If the format string is more complex (padding, prefix, suffix), implement the formatting
logic in a Compose action using the actual format string value from App Config.

The format expression is UNKNOWN until AppConfig.csv is available.

---

## Max retries

The flow reads max retries from App Config if the column exists.
If the column is missing or the value is 0, the flow uses `5` as the default.

Implementation pattern in flow:
```
if(
  or(empty(first(body('Get_items')?['value'])?['UNKNOWN_MaxRetriesColumn']),
     equals(int(first(body('Get_items')?['value'])?['UNKNOWN_MaxRetriesColumn']), 0)),
  5,
  int(first(body('Get_items')?['value'])?['UNKNOWN_MaxRetriesColumn'])
)
```

---

## What happens if App Config row is not found

If the Get items action returns 0 results (no active registry book row):

- The flow immediately returns:
  ```json
  {
    "success": false,
    "errorCode": "REGISTRY_BOOK_NOT_FOUND",
    "message": "Registry book not found in App Config."
  }
  ```
- CF_DocCentralV3_LogEvent is called with severity=Error, errorCode=REGISTRY_BOOK_NOT_FOUND.
- The parent flow (CF_DocCentralV3_CreateDocument) must treat this as a terminal failure.

---

## App Config list identity

| Property | Value |
|---|---|
| List display name | UNKNOWN — use EV_DocCentralV3_lstAppConfig |
| Environment variable | EV_DocCentralV3_lstAppConfig |
| Connection reference | CR_DocCentralV3_SharePoint |

The flow accesses the list exclusively via the environment variable — never hardcode the list name.

---

## Action to update once AppConfig.csv is available

1. Fill in all UNKNOWN values in the column mapping table above.
2. Update the filter query in Step 5a of `implementation-steps.md`.
3. Update the Set variable expressions in Step 5b of `implementation-steps.md`.
4. Update the PATCH body in Step 5c-iii of `implementation-steps.md`.
5. Update the format string expression in Step 5c-ii of `implementation-steps.md`.
