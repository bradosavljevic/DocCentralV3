# Implementation steps: CF_DocCentralV3_ArchiveDocument

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- App Config list populated with valid archive signs (UNKNOWN key — must be resolved before testing Step 3).
- All Svi predmeti internal column names confirmed (Stanje, ArhivskiZnak, Arhivirano, Arhivirao).
- ETag pattern understood — see `PACode/flows/CF_DocCentralV3_GenerateRegistryNumber/concurrency-etag-pattern.md`.
- Archive rules understood — see `PACode/flows/archive-year-rules.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_ArchiveDocument`.
3. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| documentsJson | Text | `triggerBody()?['text_documentsJson']` |
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

`documentsJson` is a JSON array string. Example value:
```json
[{"documentItemId":42,"delovodniBroj":"5/2026","arhivskiZnak":"1.1"},{"documentItemId":43,"delovodniBroj":"6/2026","arhivskiZnak":"2.3"}]
```

The Canvas App serializes the array with `JSON()` before calling the flow.

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varDocumentsArray | Array | `json(triggerBody()?['text_documentsJson'])` |
| varValidArchiveSigns | Array | `[]` |
| varArchivedCount | Integer | `0` |
| varFailedCount | Integer | `0` |
| varFailuresJson | String | `'[]'` |
| varItemETag | String | `''` |
| varCurrentStanje | String | `''` |

Do NOT use native Array variables for varFailuresJson — use String and parse/build with json()/string() to accumulate objects.
See `PACode/flows/approval-runtime-json-pattern.md` for the append pattern.

---

## Step 4 — Validate input: documents array not empty

Add **Condition**: `equals(length(variables('varDocumentsArray')), 0)`

Yes:
1. Call CF_DocCentralV3_LogEvent — Warning, NO_DOCUMENTS_PROVIDED.
2. Respond failure (NO_DOCUMENTS_PROVIDED).
3. **Terminate Succeeded.**

---

## Step 5 — Read valid archive signs from App Config

Add **SharePoint — Get items**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter: UNKNOWN — filter by archive signs category key (UNKNOWN column and value)
- Top: 500 (get all valid signs)

Rename: `Get_ValidArchiveSigns`

Add **Compose** — `Extract_ArchiveSigns`:
```
body('Get_ValidArchiveSigns')?['value']
```

Set `varValidArchiveSigns` = `outputs('Extract_ArchiveSigns')`

If the Get items returns 0 rows (no signs configured):
- Log Warning (archive signs not configured in App Config).
- Continue — no per-document validation will pass if signs are required, but do not block the run globally.

---

## Step 6 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `ArchiveDocument` / eventCategory: `Archive`
- severity: `Info` / status: `Started`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Pokretanje arhiviranja ', string(length(variables('varDocumentsArray'))), ' dokumenata.')`

---

## Step 7 — Try scope

### 7a — Apply to each document

Add **Apply to each**: input = `variables('varDocumentsArray')`

Inside the loop, add a **Scope** named `Scope_ProcessDocument` — this allows per-document error capture without triggering the global Catch.

Configure the scope's Run after to include Failed (so the loop continues on per-document errors).

#### 7a-i — Validate archive sign per document

Add **Compose** — `Get_CurrentDoc`:
```
item()
```

Add **Compose** — `Get_CurrentArhivskiZnak`:
```
outputs('Get_CurrentDoc')?['arhivskiZnak']
```

Add **Compose** — `Extract_ValidSignValues`:
```
body('Get_ValidArchiveSigns')?['value']
```

To check if the archive sign is in the valid list, use:
```
contains(
  map(outputs('Extract_ValidSignValues'), item()?['UNKNOWN_ArhivskiZnak_Value']),
  outputs('Get_CurrentArhivskiZnak')
)
```

(Replace `UNKNOWN_ArhivskiZnak_Value` with the actual internal column name once known.)

Add **Condition**: sign not valid → `equals(false, <contains result>)`

Yes (invalid sign):
1. Add **Compose** — `Build_FailureEntry_InvalidSign`:
   ```
   {
     "documentItemId": outputs('Get_CurrentDoc')?['documentItemId'],
     "delovodniBroj": outputs('Get_CurrentDoc')?['delovodniBroj'],
     "errorCode": "INVALID_ARCHIVE_SIGN",
     "message": concat('Arhivski znak "', outputs('Get_CurrentArhivskiZnak'), '" nije dozvoljen.')
   }
   ```
2. Add **Compose** — `Append_Failure_InvalidSign`:
   ```
   concat(json(variables('varFailuresJson')), array(outputs('Build_FailureEntry_InvalidSign')))
   ```
3. Set `varFailuresJson` = `string(outputs('Append_Failure_InvalidSign'))`
4. Set `varFailedCount` = `add(variables('varFailedCount'), 1)`
5. **Terminate Succeeded** (exit this scope iteration — outer Apply to each continues).

Note: Use a **Terminate** action inside the scope. In Power Automate, terminating a scope is done by configuring subsequent actions "Run after: is skipped" — or use a nested condition structure so the rest of the scope does not run after a failure branch.

#### 7a-ii — Read document item from Svi predmeti

Add **SharePoint — Get item**:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- ID: `outputs('Get_CurrentDoc')?['documentItemId']`

Rename: `Get_SviPredmeti_Item_ForArchive`

Configure **Run after**: succeeded.
Add error handling: if action status is 404 (item not found):

Check `outputs('Get_SviPredmeti_Item_ForArchive')?['statusCode']` = 404 in the condition after the action.

On 404:
1. Build failure entry: DOC_NOT_FOUND.
2. Append to varFailuresJson.
3. Increment varFailedCount.
4. Skip remaining actions in this scope iteration.

Set variable actions:
- `varCurrentStanje` = `outputs('Get_SviPredmeti_Item_ForArchive')?['body']?['UNKNOWN_Stanje']`
- `varItemETag` = `outputs('Get_SviPredmeti_Item_ForArchive')?['body']?['@odata.etag']`

#### 7a-iii — Validate Stanje = Zavedeno

Add **Condition**: `not(equals(variables('varCurrentStanje'), 'Zavedeno'))`

Yes (wrong status):
1. Build failure entry: INVALID_STATUS_FOR_ARCHIVE.
   Message: `concat('Dokument nije u statusu Zavedeno. Trenutni status: ', variables('varCurrentStanje'))`
2. Append to varFailuresJson. Increment varFailedCount. Skip remaining.

#### 7a-iv — PATCH document to Arhivirano (If-Match)

Obtain request digest first if not already obtained in this loop iteration.
(To avoid an extra HTTP call per document, obtain request digest once outside the loop in Step 5.5 — see Power Automate build notes.)

Add **SharePoint — Send an HTTP Request** — PATCH:
- Site: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Method: POST
- Uri: `concat("_api/web/lists/GetByTitle('", @parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'), "')/items(", outputs('Get_CurrentDoc')?['documentItemId'], ")")`
- Headers:
  - `Accept`: `application/json;odata=verbose`
  - `Content-Type`: `application/json;odata=verbose`
  - `IF-MATCH`: `variables('varItemETag')`
  - `X-HTTP-Method`: `MERGE`
  - `X-RequestDigest`: `variables('varRequestDigest')`
- Body (all UNKNOWN internal column names — replace before building):
```json
{
  "__metadata": { "type": "SP.Data.SviPredmetiListItem" },
  "UNKNOWN_Stanje": "Arhivirano",
  "UNKNOWN_ArhivskiZnak": "@{outputs('Get_CurrentDoc')?['arhivskiZnak']}",
  "UNKNOWN_Arhivirano": "@{utcNow()}",
  "UNKNOWN_Arhivirao": "@{triggerBody()?['text_initiatorEmail']}"
}
```

Rename: `PATCH_SviPredmeti_Archive`

**412 handling:**

Add **Condition** after PATCH: `equals(outputs('PATCH_SviPredmeti_Archive')?['statusCode'], 412)`

Yes (412):
1. Add **SharePoint — Get item** (re-read):
   - Rename: `Get_SviPredmeti_Item_Retry`
   - Same ID as above.
2. Add **Compose** — `Get_RetryStanje`:
   ```
   outputs('Get_SviPredmeti_Item_Retry')?['body']?['UNKNOWN_Stanje']
   ```
3. Condition: `equals(outputs('Get_RetryStanje'), 'Arhivirano')`
   - Yes: treat as success — increment varArchivedCount, log per-document success, continue.
   - No: add to varFailuresJson (CONCURRENT_UPDATE), increment varFailedCount, continue.

**Other PATCH error handling:**

Add **Condition**: `not(or(equals(outputs('PATCH_SviPredmeti_Archive')?['statusCode'], 200), equals(outputs('PATCH_SviPredmeti_Archive')?['statusCode'], 204)))`

Yes:
1. Add to varFailuresJson (ARCHIVE_UPDATE_FAILED).
2. Increment varFailedCount. Continue.

#### 7a-v — Log per-document success and increment count

Call CF_DocCentralV3_LogEvent:
- eventType: `ArchiveDocument` / severity: `Info` / status: `Success`
- documentItemId: `outputs('Get_CurrentDoc')?['documentItemId']`
- delovodniBroj: `outputs('Get_CurrentDoc')?['delovodniBroj']`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Dokument ', outputs('Get_CurrentDoc')?['delovodniBroj'], ' je arhiviran. Arhivski znak: ', outputs('Get_CurrentDoc')?['arhivskiZnak'], '.')`

Set `varArchivedCount` = `add(variables('varArchivedCount'), 1)`

---

## Step 7.5 — Request digest (outside the loop)

Obtain the request digest once before entering the Apply to each loop. It is valid for 30 minutes — sufficient for any realistic batch.

Add **SharePoint — Send an HTTP Request**:
- Method: POST
- Uri: `_api/contextinfo`
- Headers: `Accept: application/json;odata=verbose`

Rename: `Get_RequestDigest`

Add **Compose** — `Extract_RequestDigest`:
```
body('Get_RequestDigest')?['d']?['GetContextWebInformation']?['FormDigestValue']
```

Initialize `varRequestDigest` (String) in Step 3 and set it here:
Set `varRequestDigest` = `outputs('Extract_RequestDigest')`

Place this step between Step 6 (Log Started) and Step 7 (Apply to each).

---

## Step 8 — Log overall result

After the Apply to each:

Add **Condition**: `equals(variables('varFailedCount'), 0)`

Yes (all succeeded):
- Call LogEvent: severity = `Info`, status = `Success`
- message: `concat('Arhiviranje završeno. Uspešno: ', string(variables('varArchivedCount')), ', Neuspešno: 0.')`

No (some or all failed):
- Sub-condition: `equals(variables('varArchivedCount'), 0)` (all failed)
  - Yes: severity = `Error`
  - No: severity = `Warning` (partial)
- Call LogEvent: status = `Failed` if all failed, `Success` with Warning if partial.
- message: `concat('Arhiviranje završeno. Uspešno: ', string(variables('varArchivedCount')), ', Neuspešno: ', string(variables('varFailedCount')), '.')`

---

## Step 9 — Respond

**Respond to a PowerApp or flow** with 6 outputs:

| Output | Value |
|---|---|
| success | `equals(variables('varFailedCount'), 0)` |
| message | `if(equals(variables('varFailedCount'), 0), 'Dokumenti su uspešno arhivirani.', 'Deo dokumenata nije arhiviran.')` |
| archivedCount | `variables('varArchivedCount')` |
| failedCount | `variables('varFailedCount')` |
| failures | `variables('varFailuresJson')` |
| correlationId | `variables('varCorrelationId')` |

---

## Step 10 — Catch scope

Configure Run after: failed, timed out (on the Try scope if using a Try scope, or directly on key actions).

Call CF_DocCentralV3_LogEvent:
- eventType: `ArchiveDocument` / severity: `Error` / status: `Failed`
- errorCode: `ARCHIVE_DOCUMENT_FAILED`
- errorMessage: `result('Try')?[0]?['error']?['message']`
- correlationId: `variables('varCorrelationId')`

Respond failure:
- success: false
- message: `Greška pri arhiviranju dokumenata.`
- archivedCount: `variables('varArchivedCount')` (partial count if available)
- failedCount: `variables('varFailedCount')`
- failures: `variables('varFailuresJson')`
- correlationId: `variables('varCorrelationId')`

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_ArchiveDocument`
- [ ] Trigger: Power Apps (V2) with 3 inputs (documentsJson as Text)
- [ ] All 8 variables initialized
- [ ] Empty documents array validated before loop
- [ ] App Config archive signs read once before loop
- [ ] Request digest obtained once before loop
- [ ] Log Started before loop
- [ ] Apply to each iterates over varDocumentsArray
- [ ] Per document: validate archive sign, get item, validate Stanje, PATCH
- [ ] PATCH uses IF-MATCH with varItemETag and X-HTTP-Method: MERGE
- [ ] 412: re-read, check Stanje, idempotent if already Arhivirano
- [ ] Failures accumulated in varFailuresJson (not native Array variable)
- [ ] varArchivedCount and varFailedCount incremented correctly
- [ ] Per-document log on success
- [ ] Overall log after loop (Info / Warning / Error by result)
- [ ] Response includes failures JSON string
- [ ] Catch scope: log Error, respond failure
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow inside DocCentralV3 solution
