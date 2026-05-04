# Implementation steps: CF_DocCentralV3_SendForApproval

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- CF_DocCentralV3_AssignPermissions published.
- Svi predmeti list exists with all approval runtime columns confirmed.
- App Config list has ProcesConfig entry (at least one document type with steps).
- AuditLog list exists.
- Connection references connected: CR_DocCentralV3_SharePoint, CR_DocCentralV3_Outlook, CR_DocCentralV3_Office365Groups.
- Environment variables registered: EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog.
- Request digest pattern understood — see `power-automate-build-notes.md`.
- JSON array manipulation pattern understood — see `PACode/flows/approval-runtime-json-pattern.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_SendForApproval`.
3. Trigger placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| documentItemId | Number | `triggerBody()?['decimal_documentItemId']` |
| delovodniBroj | Text | `triggerBody()?['text_delovodniBroj']` |
| documentType | Text | `triggerBody()?['text_documentType']` |
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| initiatorDisplayName | Text | `triggerBody()?['text_initiatorDisplayName']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varDocumentItemId | Integer | `int(triggerBody()?['decimal_documentItemId'])` |
| varDocETag | String | `''` |
| varApprovalSteps | Array | `[]` |
| varKoraciJson | String | `'[]'` |
| varIstorijaJson | String | `'[]'` |
| varCurrentRound | Integer | `1` |
| varStep1 | Object | `{}` |
| varRecipients | Array | `[]` |

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `SendForApproval` / eventCategory: `Approval`
- severity: `Info` / status: `Started`
- documentItemId: `variables('varDocumentItemId')`
- delovodniBroj: `triggerBody()?['text_delovodniBroj']`
- userEmail: `triggerBody()?['text_initiatorEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `Pokretanje procesa odobrenja.`

---

## Step 5 — Try scope

### 5a — Get Svi predmeti item

Add **SharePoint — Get item**:
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- ID: `variables('varDocumentItemId')`

Rename: `Get_SviPredmeti_Item`

On 404 / action failure → Log DOCUMENT_NOT_FOUND / Error, Respond failure, Terminate Succeeded.

Store:
- varDocETag = `outputs('Get_SviPredmeti_Item')?['body']?['@odata.etag']`
- varStanje = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_Stanje']`
- varExistingIstorija = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_IstorijaOdobravanjaJson']`

### 5b — Pre-condition checks

**Check 1 — ALREADY_IN_APPROVAL:**
Condition: `equals(variables('varStanje'), 'U odobravanju')`
Yes → Log warning / ALREADY_IN_APPROVAL, Respond failure, Terminate Succeeded.

**Check 2 — DOCUMENT_ARCHIVED:**
Condition: `equals(variables('varStanje'), 'Arhivirano')`
Yes → Log warning / DOCUMENT_ARCHIVED, Respond failure, Terminate Succeeded.

**Check 3 — Valid starting states:**
Allowed: `Zavedeno`, `Odbijeno`. Any other Stanje value (e.g. `Odobreno`) should also be blocked.
Add: Condition `not(or(equals(varStanje, 'Zavedeno'), equals(varStanje, 'Odbijeno')))` → Log warning / STATUS_UPDATE_FAILED, Respond failure, Terminate Succeeded.

### 5c — Read ProcesConfig from App Config

Add **SharePoint — Get items**:
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')`
- Filter: UNKNOWN — filter for ProcesConfig entries matching `documentType` input
- Top: 1

Rename: `Get_AppConfig_ProcesConfig`

If empty → try default ProcesConfig (second Get items with UNKNOWN default key).
If still empty → Log NO_PROCESS_CONFIG / Error, Respond failure, Terminate Succeeded.

Parse steps from result:
```
varApprovalSteps = json(first(body('Get_AppConfig_ProcesConfig')?['value'])?['UNKNOWN_StepsColumn'])?['steps']
```
(Adjust expression based on actual ProcesConfig column structure — UNKNOWN.)

If `length(varApprovalSteps) = 0` → Log NO_APPROVAL_STEPS / Error, Respond failure, Terminate Succeeded.

### 5d — Determine current round and build IstorijaJson

**If resubmission** (varStanje was `Odbijeno` and varExistingIstorija is not empty/`[]`):

1. Parse existing IstorijaOdobravanjaJson.
2. Count `RoundReset` entries to determine previous round count:
   ```
   varCurrentRound = add(length(filter(json(variables('varExistingIstorija')), equals(item()?['outcome'], 'RoundReset'))), 1)
   ```
3. Build RoundReset marker entry (see `approval-runtime-json-pattern.md`).
4. Append marker to existing history array.
5. Set varIstorijaJson = serialized extended array.

**If first submission** (varExistingIstorija is empty or `[]`):
- varCurrentRound = 1
- varIstorijaJson = `'[]'`

### 5e — Build KoraciOdobravanjaJson

Compose the full step array with status = `Pending` on all steps.

Add **Compose** — `Build_KoraciJson`:
Build JSON array from varApprovalSteps, adding Pending status fields:

Use Apply to each over varApprovalSteps with Append to array variable:
```
varKoraciArray += {
  stepNumber: item()?['stepNumber'],
  assigneeType: item()?['assigneeType'],
  assigneeEmail: item()?['assigneeEmail'],
  assigneeGroupId: item()?['assigneeGroupId'],
  status: 'Pending',
  resolvedBy: null,
  resolvedAt: null,
  comments: null
}
```

Then: varKoraciJson = `string(variables('varKoraciArray'))`

Rename loop: `Build_KoraciOdobravanjaJson_Loop`

### 5f — Extract step 1

Add **Compose** — `Get_Step1`:
```
first(filter(variables('varApprovalSteps'), equals(item()?['stepNumber'], 1)))
```

Store result as varStep1 (use Set variable with the compose output).

### 5g — Obtain request digest

Add **SharePoint — Send an HTTP Request**:
- Method: POST
- Uri: `_api/contextinfo`
- Headers: `Accept: application/json;odata=nometadata`

Rename: `Get_RequestDigest`

Extract: `varRequestDigest = outputs('Get_RequestDigest')?['body']?['FormDigestValue']`

### 5h — PATCH Svi predmeti with approval state (If-Match)

Add **SharePoint — Send an HTTP Request** — PATCH:
- Method: POST
- Uri: concat expression for `_api/web/lists/getbytitle(…)/items(varDocumentItemId)`
- Headers:
  - `IF-MATCH: variables('varDocETag')`
  - `X-HTTP-Method: MERGE`
  - `X-RequestDigest: variables('varRequestDigest')`
  - `Accept: application/json;odata=nometadata`
  - `Content-Type: application/json;odata=nometadata`
- Body: JSON with all 7 approval fields (internal names UNKNOWN — see `field-mapping.md`):
  - TrenutniOdobravalacEmail: `if(equals(variables('varStep1')?['assigneeType'], 'User'), variables('varStep1')?['assigneeEmail'], '')`
  - TrenutnaGrupaOdobravanjaId: `if(equals(variables('varStep1')?['assigneeType'], 'Group'), variables('varStep1')?['assigneeGroupId'], '')`

Rename: `PATCH_SviPredmeti_ApprovalState`

**412 handling:**
Add Condition: `equals(outputs('PATCH_SviPredmeti_ApprovalState')?['statusCode'], 412)`
Yes:
1. Re-read item (Get_SviPredmeti_Item again).
2. Check Stanje — if now `U odobravanju` → return ALREADY_IN_APPROVAL.
3. Otherwise: refresh ETag and retry PATCH once.
4. If second attempt also fails: Log STATUS_UPDATE_FAILED / Error, Respond failure, Terminate Succeeded.

**Other failures:**
Condition: status code not in [200, 204] after retry → STATUS_UPDATE_FAILED.

### 5i — Resolve notification recipients for step 1

Condition: `equals(variables('varStep1')?['assigneeType'], 'User')`

Yes (User step):
- Set varRecipients = `array(variables('varStep1')?['assigneeEmail'])`

No (Group step):
- Add **Office365Groups — List group members**:
  - Group Id: `variables('varStep1')?['assigneeGroupId']`
  Rename: `List_Step1_GroupMembers`
- Set varRecipients = extract email list from group members response:
  ```
  body('List_Step1_GroupMembers')?['value']
  ```
  (Map to email strings using Select action — see `power-automate-build-notes.md`.)

### 5j — Grant step 1 approver access

Call CF_DocCentralV3_AssignPermissions:
- sviPredmetiItemId: `variables('varDocumentItemId')`
- documentLibraryFileIds: `'[]'`
- initiatorEmail: `triggerBody()?['text_initiatorEmail']`
- documentType: `triggerBody()?['text_documentType']`
- orgUnitGroupId: `''`
- additionalUserEmails: `if(equals(varStep1?['assigneeType'], 'User'), string(array(varStep1?['assigneeEmail'])), '[]')`
- additionalGroupIds: `if(equals(varStep1?['assigneeType'], 'Group'), string(array(varStep1?['assigneeGroupId'])), '[]')`
- correlationId: `variables('varCorrelationId')`

Permission failure is non-fatal — log Warning and continue.

### 5k — Send notification email to step 1 recipients

Add **Apply to each** over `variables('varRecipients')`:

Inside loop:
- Add **Outlook — Send an email (V2)**:
  - To: current item email
  - Subject: `concat('DocCentral: Zahtev za odobrenje dokumenta ', triggerBody()?['text_delovodniBroj'])`
  - Body: UNKNOWN template. Placeholder: include delovodniBroj, initiator display name, instruction to open Canvas App to approve.

Email failure per recipient is non-fatal — log Warning per failed recipient, continue loop.

Rename loop: `Notify_Step1_Recipients`

### 5l — Log Success

Call CF_DocCentralV3_LogEvent:
- eventType: `SendForApproval` / severity: `Info` / status: `Success`
- documentItemId: `variables('varDocumentItemId')`
- delovodniBroj: `triggerBody()?['text_delovodniBroj']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Dokument ', triggerBody()?['text_delovodniBroj'], ' je poslat na odobrenje. Koraci: ', string(length(variables('varApprovalSteps'))), '.')`

### 5m — Respond success

Add **Respond to a PowerApp or flow**:
- success: `true`
- message: `Dokument je poslat na odobrenje.`
- totalSteps: `length(variables('varApprovalSteps'))`
- firstApprover: `if(equals(varStep1?['assigneeType'], 'User'), varStep1?['assigneeEmail'], varStep1?['assigneeGroupId'])`
- correlationId: `variables('varCorrelationId')`
- errorCode: `''`

---

## Step 6 — Catch scope

Configure Run after: failed, timed out.

Inside:
1. Call CF_DocCentralV3_LogEvent:
   - eventType: `SendForApproval` / severity: `Error` / status: `Failed`
   - errorCode: `SEND_FOR_APPROVAL_FAILED`
   - errorMessage: `result('Try')?[0]?['error']?['message']`
2. Respond failure (6 outputs).

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_SendForApproval`
- [ ] Trigger: Power Apps (V2) with 6 inputs
- [ ] All 9 variables initialized; varCorrelationId auto-GUID
- [ ] Log Started before Try scope
- [ ] Get item: 404 handled (DOCUMENT_NOT_FOUND)
- [ ] Stanje pre-conditions: ALREADY_IN_APPROVAL, DOCUMENT_ARCHIVED, other invalid states blocked
- [ ] ProcesConfig read from App Config; NO_PROCESS_CONFIG and NO_APPROVAL_STEPS handled
- [ ] Resubmission: IstorijaJson extended with RoundReset marker; varCurrentRound incremented
- [ ] First submission: varIstorijaJson = `'[]'`, varCurrentRound = 1
- [ ] KoraciOdobravanjaJson built with all steps Pending
- [ ] Request digest obtained before PATCH
- [ ] PATCH uses If-Match header with varDocETag
- [ ] 412 handled: re-read, re-check, retry once
- [ ] Group step: group members listed via Office365Groups
- [ ] AssignPermissions called (non-fatal on failure)
- [ ] Email sent to recipients (non-fatal per recipient)
- [ ] Log Success before Respond success
- [ ] Catch scope: Log Error, Respond failure
- [ ] Catch never calls CF_DocCentralV3_SendForApproval
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow is inside DocCentralV3 solution
