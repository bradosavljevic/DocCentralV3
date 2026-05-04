# Power Automate build notes: CF_DocCentralV3_ArchiveDocument

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_ArchiveDocument |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog |

---

## documentsJson parsing

The trigger receives the documents array as a JSON string (Text type).
Parse immediately after trigger:

```
json(triggerBody()?['text_documentsJson'])
```

Use null-safe guard before parsing:
```
if(empty(triggerBody()?['text_documentsJson']), '[]', triggerBody()?['text_documentsJson'])
```

Assign to `varDocumentsArray` (Array variable) so Apply to each can iterate directly.

---

## Apply to each: reference current item

Inside the loop, the current document object is accessed with:
```
item()
```

To avoid reference errors in nested actions, add a Compose at the top of the loop:
```
[Compose: Get_CurrentDoc]
item()
```

Then reference fields as:
```
outputs('Get_CurrentDoc')?['documentItemId']
outputs('Get_CurrentDoc')?['delovodniBroj']
outputs('Get_CurrentDoc')?['arhivskiZnak']
```

---

## Accumulating failures — String-based pattern

Do not use native Array variables for the failures collection. Power Automate's Append to array variable action appends strings or scalars, not objects — the result is unreliable for JSON objects.

Use the same pattern as IstorijaOdobravanjaJson in the approval flows:
```
[Compose: Build_FailureEntry]
{
  "documentItemId": outputs('Get_CurrentDoc')?['documentItemId'],
  "delovodniBroj": outputs('Get_CurrentDoc')?['delovodniBroj'],
  "errorCode": "INVALID_ARCHIVE_SIGN",
  "message": "<error message>"
}

[Compose: Append_Failure]
concat(json(variables('varFailuresJson')), array(outputs('Build_FailureEntry')))

[Set variable: varFailuresJson]
string(outputs('Append_Failure'))
```

This is the same pattern used for `IstorijaOdobravanjaJson` — object expression (not string concat) to avoid quote escaping issues.

---

## Request digest — obtain once outside the loop

The request digest is valid for approximately 30 minutes.
Obtain it once before the Apply to each loop to avoid one extra HTTP call per document.

```
[Send an HTTP Request: Get_RequestDigest]
Method: POST
Uri: _api/contextinfo
Headers: Accept: application/json;odata=verbose

[Compose: Extract_RequestDigest]
body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']

[Set variable: varRequestDigest]
outputs('Extract_RequestDigest')
```

---

## Per-document scope — skipping remaining actions after failure

Power Automate does not have a per-iteration `continue` or `break`.
To skip remaining actions in the current loop iteration after a per-document failure:

**Option A** — nested conditions:
After each validation step, wrap all remaining actions in the `No` branch of the validation condition.
This means validation conditions nest deeper for each check.

**Option B** — Scope with Terminate:
Place per-document actions in a child Scope named `Scope_ProcessDocument`.
Use a `Terminate` action (Status: Succeeded) inside the scope to exit early.
Remaining actions after the child scope continue normally (loop moves to next item).

Option B is cleaner for 3+ sequential validations. Use it here.

---

## App Config archive signs — contains check

Once `varValidArchiveSigns` holds the App Config result array:

```
[Compose: Extract_ValidSignValues]
map(body('Get_ValidArchiveSigns')?['value'], item()?['UNKNOWN_SignValue'])
```

(Replace `UNKNOWN_SignValue` with the actual internal column name for the sign code.)

Check if current document's sign is valid:
```
contains(outputs('Extract_ValidSignValues'), outputs('Get_CurrentArhivskiZnak'))
```

Note: `contains()` is case-sensitive. If signs are stored in App Config with mixed case, normalize both with `toLower()`:
```
contains(
  map(body('Get_ValidArchiveSigns')?['value'], toLower(item()?['UNKNOWN_SignValue'])),
  toLower(outputs('Get_CurrentArhivskiZnak'))
)
```

---

## 412 handling — idempotency check

Unlike approval flows (which always treat 412 as a conflict), ArchiveDocument has an idempotency rule:
if the re-read shows Stanje = `Arhivirano`, count the document as archived (same intent already achieved).

Pattern:
```
[After PATCH_SviPredmeti_Archive]
Condition: statusCode = 412
Yes:
  [Get_SviPredmeti_Item_Retry] — re-read item
  [Compose: Get_RetryStanje]
  outputs('Get_SviPredmeti_Item_Retry')?['body']?['UNKNOWN_Stanje']

  Condition: equals(outputs('Get_RetryStanje'), 'Arhivirano')
  Yes → count as success (increment varArchivedCount, log success)
  No → append CONCURRENT_UPDATE failure (increment varFailedCount)
```

---

## Get item — 404 pattern

SharePoint Get item returns 404 inside the action's status code, but Power Automate marks the action as failed.
To catch 404:

1. Configure `Get_SviPredmeti_Item_ForArchive` **Run after** to include Failed.
2. Check `outputs('Get_SviPredmeti_Item_ForArchive')?['statusCode']` = 404.
3. If 404: append DOC_NOT_FOUND failure and skip remaining steps for this document.
4. If other error: append ARCHIVE_UPDATE_FAILED failure and skip.

Alternatively, use a Scope wrapping the Get item and configure its Run after to handle failed state.

---

## Incrementing integer counters in a loop

Power Automate's `Set variable` to `add(variable, 1)` works correctly inside Apply to each.
Do not use `Increment variable` — it exists but behaves the same as Set + add.

```
[Set variable: varArchivedCount]
add(variables('varArchivedCount'), 1)
```

---

## Overall success determination

After the loop:
```
[Compose: IsSuccess]
equals(variables('varFailedCount'), 0)
```

Use `outputs('IsSuccess')` in the Respond action's `success` output.

---

## Response: failures as string

The `failures` output is a String (not Array) because Power Apps V2 trigger outputs do not support native Array type.
The Canvas App receives the JSON string and parses it with `ParseJSON()`.

The flow always returns `variables('varFailuresJson')` — it starts as `'[]'` and accumulates entries.

---

## Action naming convention

| Purpose | Name |
|---|---|
| Get valid archive signs | `Get_ValidArchiveSigns` |
| Extract sign values | `Extract_ArchiveSigns`, `Extract_ValidSignValues` |
| Get request digest | `Get_RequestDigest`, `Extract_RequestDigest` |
| Current doc reference | `Get_CurrentDoc`, `Get_CurrentArhivskiZnak` |
| Per-document scope | `Scope_ProcessDocument` |
| Get Svi predmeti item | `Get_SviPredmeti_Item_ForArchive` |
| Get Svi predmeti item (retry) | `Get_SviPredmeti_Item_Retry` |
| Retry Stanje check | `Get_RetryStanje` |
| Failure entry builds | `Build_FailureEntry_InvalidSign`, `Build_FailureEntry_NotFound`, etc. |
| Failure append | `Append_Failure_*` |
| PATCH | `PATCH_SviPredmeti_Archive` |

---

## Checklist

- [ ] documentsJson parsed with null-safe guard
- [ ] Apply to each iterates over varDocumentsArray (Array variable)
- [ ] Archive signs read once before loop
- [ ] Request digest obtained once before loop
- [ ] Per-document scope allows early exit without failing the loop
- [ ] varFailuresJson accumulated using string(concat(json(...), array(...))) pattern
- [ ] PATCH uses IF-MATCH with item ETag and X-HTTP-Method: MERGE
- [ ] 412: re-read, check Stanje, idempotent if already Arhivirano
- [ ] varArchivedCount and varFailedCount incremented correctly per branch
- [ ] Per-document log on success
- [ ] Overall log after loop with correct severity
- [ ] failures output is varFailuresJson (string) — not a native array
- [ ] Flow inside DocCentralV3 solution
