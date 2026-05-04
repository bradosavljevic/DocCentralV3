# Implementation steps: CF_DocCentralV3_ProcessApprovalResponse

## Prerequisites

- CF_DocCentralV3_LogEvent published.
- CF_DocCentralV3_AssignPermissions published.
- CF_DocCentralV3_SendForApproval published (needed for end-to-end testing).
- All Svi predmeti internal column names confirmed.
- JSON array manipulation pattern understood — see `PACode/flows/approval-runtime-json-pattern.md`.
- ETag pattern understood — see `PACode/flows/CF_DocCentralV3_GenerateRegistryNumber/concurrency-etag-pattern.md`.

---

## Step 1 — Create the flow

1. DocCentralV3 solution → New → Cloud flow → Instant.
2. Name: `CF_DocCentralV3_ProcessApprovalResponse`.
3. Trigger: placeholder — replace in Step 2.

---

## Step 2 — Trigger: Power Apps (V2)

Add inputs:

| Input name | Type | Internal reference |
|---|---|---|
| documentItemId | Number | `triggerBody()?['decimal_documentItemId']` |
| outcome | Text | `triggerBody()?['text_outcome']` |
| comments | Text | `triggerBody()?['text_comments']` |
| responderEmail | Text | `triggerBody()?['text_responderEmail']` |
| responderDisplayName | Text | `triggerBody()?['text_responderDisplayName']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |

---

## Step 3 — Initialize variables

| Variable | Type | Initial value |
|---|---|---|
| varCorrelationId | String | `if(empty(triggerBody()?['text_correlationId']), guid(), triggerBody()?['text_correlationId'])` |
| varDocumentItemId | Integer | `int(triggerBody()?['decimal_documentItemId'])` |
| varDocETag | String | `''` |
| varStanje | String | `''` |
| varTrenutniKorak | Integer | `0` |
| varUkupnoKoraka | Integer | `0` |
| varOdobravalacEmail | String | `''` |
| varGrupaId | String | `''` |
| varKoraciJson | String | `'[]'` |
| varIstorijaJson | String | `'[]'` |
| varInicijatorEmail | String | `''` |
| varDelovodniBroj | String | `''` |
| varNextAction | String | `''` |
| varCurrentRound | Integer | `1` |
| varUpdatedKoraciJson | String | `'[]'` |
| varUpdatedIstorijaJson | String | `'[]'` |
| varNaredniKorakObj | Object | `{}` |
| varRequestDigest | String | `''` |

---

## Step 4 — Log Started

Call CF_DocCentralV3_LogEvent:
- eventType: `ProcessApprovalResponse` / eventCategory: `Approval`
- severity: `Info` / status: `Started`
- documentItemId: `variables('varDocumentItemId')`
- userEmail: `triggerBody()?['text_responderEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `Obrada odgovora odobravača.`

---

## Step 5 — Try scope

### 5a — Validate outcome input

Condition: `not(or(equals(triggerBody()?['text_outcome'], 'Approved'), equals(triggerBody()?['text_outcome'], 'Rejected')))`
Yes → Respond INVALID_OUTCOME, Terminate Succeeded.

### 5b — Get Svi predmeti item

Add **SharePoint — Get item**:
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')`
- ID: `variables('varDocumentItemId')`

Rename: `Get_SviPredmeti_Item`

On 404 → Respond DOCUMENT_NOT_FOUND, Terminate Succeeded.

Set all read variables from item body:
- varDocETag = `outputs('Get_SviPredmeti_Item')?['body']?['@odata.etag']`
- varStanje = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_Stanje']`
- varTrenutniKorak = `int(outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_TrenutniKorakOdobravanja'])`
- varUkupnoKoraka = `int(outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_UkupnoKorakaOdobravanja'])`
- varOdobravalacEmail = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_TrenutniOdobravalacEmail']`
- varGrupaId = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_TrenutnaGrupaOdobravanjaId']`
- varKoraciJson = `if(empty(outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_KoraciOdobravanjaJson']), '[]', outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_KoraciOdobravanjaJson'])`
- varIstorijaJson = `if(empty(outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_IstorijaOdobravanjaJson']), '[]', outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_IstorijaOdobravanjaJson'])`
- varInicijatorEmail = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_InicijatorEmail']`
- varDelovodniBroj = `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN_DelovodniBroj']`

### 5c — Validate Stanje = U odobravanju

Condition: `not(equals(variables('varStanje'), 'U odobravanju'))`
Yes → Respond NOT_IN_APPROVAL, Terminate Succeeded.

### 5d — Validate KoraciOdobravanjaJson parseable

Condition: `or(empty(variables('varKoraciJson')), equals(length(json(variables('varKoraciJson'))), 0))`
Yes → Respond APPROVAL_STATE_INVALID, Terminate Succeeded.

### 5e — Find current step object in KoraciJson

Add **Compose** — `Get_TrenutniKorakObj`:
```
first(filter(json(variables('varKoraciJson')), equals(item()?['stepNumber'], variables('varTrenutniKorak'))))
```

Add Condition: if output is null → Respond APPROVAL_STATE_INVALID, Terminate Succeeded.

### 5f — Validate current step is Pending

Add **Compose** — `Get_TrenutniKorakStatus`:
```
outputs('Get_TrenutniKorakObj')?['status']
```

Condition: `not(equals(outputs('Get_TrenutniKorakStatus'), 'Pending'))`
Yes → Respond STEP_ALREADY_RESOLVED, Terminate Succeeded.

### 5g — Authorize responder

**User step** (varOdobravalacEmail is not empty):
Condition: `not(empty(variables('varOdobravalacEmail')))`
Yes:
- Condition: `not(equals(toLower(triggerBody()?['text_responderEmail']), toLower(variables('varOdobravalacEmail'))))`
  Yes → Respond RESPONDER_NOT_AUTHORIZED, Terminate Succeeded.

**Group step** (varGrupaId is not empty):
Condition: `not(empty(variables('varGrupaId')))`
Yes:
1. Add **Office365Groups — List group members**:
   - Group Id: `variables('varGrupaId')`
   Rename: `List_CurrentGroup_Members`
2. Add **Compose** — extract email list:
   ```
   map(body('List_CurrentGroup_Members')?['value'], item()?['mail'])
   ```
3. Condition: `not(contains(outputs('Extract_GroupMember_Emails'), toLower(triggerBody()?['text_responderEmail'])))`
   Yes → Respond RESPONDER_NOT_AUTHORIZED, Terminate Succeeded.
   On API failure: log Warning, continue with audit flag.

### 5h — Compute current round

Add **Compose** — `Compute_CurrentRound`:
```
add(length(filter(json(variables('varIstorijaJson')), equals(item()?['outcome'], 'RoundReset'))), 1)
```

Set varCurrentRound = `int(outputs('Compute_CurrentRound'))`

### 5i — Update current step in KoraciOdobravanjaJson

See `field-mapping.md` and `approval-runtime-json-pattern.md` for the full expression.

Use multiple Compose actions:

1. **Compose** `Build_UpdatedCurrentStep` — setProperty chain on the current step object:
   ```
   setProperty(
     setProperty(
       setProperty(
         setProperty(outputs('Get_TrenutniKorakObj'), 'status', triggerBody()?['text_outcome']),
         'resolvedBy', triggerBody()?['text_responderEmail']
       ),
       'resolvedAt', utcNow()
     ),
     'comments', triggerBody()?['text_comments']
   )
   ```

2. **Compose** `Build_StepsBefore` — steps before current:
   ```
   filter(json(variables('varKoraciJson')), less(item()?['stepNumber'], variables('varTrenutniKorak')))
   ```

3. **Compose** `Build_StepsAfter` — steps after current:
   ```
   filter(json(variables('varKoraciJson')), greater(item()?['stepNumber'], variables('varTrenutniKorak')))
   ```

4. **Compose** `Build_UpdatedKoraciArray`:
   ```
   concat(outputs('Build_StepsBefore'), array(outputs('Build_UpdatedCurrentStep')), outputs('Build_StepsAfter'))
   ```

5. Set varUpdatedKoraciJson = `string(outputs('Build_UpdatedKoraciArray'))`

### 5j — Append to IstorijaOdobravanjaJson

Add **Compose** `Build_HistoryEntry`:
```
{
  "round": variables('varCurrentRound'),
  "stepNumber": variables('varTrenutniKorak'),
  "outcome": triggerBody()?['text_outcome'],
  "byEmail": triggerBody()?['text_responderEmail'],
  "byName": triggerBody()?['text_responderDisplayName'],
  "at": utcNow(),
  "comments": triggerBody()?['text_comments']
}
```

Add **Compose** `Build_UpdatedIstorijaArray`:
```
concat(json(variables('varIstorijaJson')), array(outputs('Build_HistoryEntry')))
```

Set varUpdatedIstorijaJson = `string(outputs('Build_UpdatedIstorijaArray'))`

### 5k — Branch on outcome

**Condition: outcome = `Rejected`**

Yes branch:
1. **Mark remaining steps Skipped** — see `field-mapping.md` for expression.
2. Set varNextAction = `'DocumentRejected'`
3. Build PATCH body (Branch A — see `field-mapping.md`).

No branch (outcome = `Approved`):

Sub-condition: Does next step exist?
```
greater(add(variables('varTrenutniKorak'), 1), variables('varUkupnoKoraka'))
```
Yes (no next step — ProcessComplete):
1. Set varNextAction = `'ProcessComplete'`
2. Build PATCH body — Branch C (Stanje = Odobreno, clear approver fields).

No (next step exists — NextStepActivated):
1. Find next step object:
   ```
   first(filter(json(variables('varKoraciJson')), equals(item()?['stepNumber'], add(variables('varTrenutniKorak'), 1))))
   ```
   Store as varNaredniKorakObj.
2. Set varNextAction = `'NextStepActivated'`
3. Build PATCH body — Branch B (advance current step fields to next step).

### 5l — Obtain request digest

Add **SharePoint — Send an HTTP Request** → POST `_api/contextinfo`.
Set varRequestDigest from response.

### 5m — PATCH Svi predmeti (If-Match)

Add **SharePoint — Send an HTTP Request** — PATCH:
- IF-MATCH: `variables('varDocETag')`
- X-HTTP-Method: MERGE
- X-RequestDigest: `variables('varRequestDigest')`
- Body: PATCH body from Step 5k (Branch A, B, or C)

Rename: `PATCH_SviPredmeti_ApprovalOutcome`

**412 handling:**
Condition: status code = 412
Yes:
1. Re-read item.
2. Check current step status in KoraciOdobravanjaJson.
3. If step no longer Pending → Respond STEP_ALREADY_RESOLVED, Terminate Succeeded.
4. Otherwise: refresh ETag, retry PATCH once.
5. Second 412: Respond STATUS_UPDATE_FAILED, Terminate Succeeded.

### 5n — Post-outcome actions

**If NextStepActivated:**
1. Resolve recipients for varNaredniKorakObj (same pattern as SendForApproval Step 5i).
2. Call CF_DocCentralV3_AssignPermissions — grant next approver access (non-fatal).
3. Send notification email to next step recipients (non-fatal).

**If ProcessComplete or DocumentRejected:**
1. Send notification email to varInicijatorEmail (non-fatal).
   - ProcessComplete subject: `concat('DocCentral: Dokument odobren - ', variables('varDelovodniBroj'))`
   - DocumentRejected subject: `concat('DocCentral: Dokument odbijen - ', variables('varDelovodniBroj'))`

### 5o — Log Success

Call CF_DocCentralV3_LogEvent:
- eventType: `ProcessApprovalResponse` / severity: `Info` / status: `Success`
- documentItemId: `variables('varDocumentItemId')`
- delovodniBroj: `variables('varDelovodniBroj')`
- userEmail: `triggerBody()?['text_responderEmail']`
- correlationId: `variables('varCorrelationId')`
- message: `concat('Korak ', string(variables('varTrenutniKorak')), ' - ', triggerBody()?['text_outcome'], ' od strane ', triggerBody()?['text_responderEmail'], '. Sledeća akcija: ', variables('varNextAction'), '.')`

### 5p — Respond success

- success: `true`
- message: `Odgovor je zabeležen.`
- nextAction: `variables('varNextAction')`
- correlationId: `variables('varCorrelationId')`
- errorCode: `''`

---

## Step 6 — Catch scope

Configure Run after: failed, timed out.

1. Call CF_DocCentralV3_LogEvent:
   - eventType: `ProcessApprovalResponse` / severity: `Error` / status: `Failed`
   - errorCode: `PROCESS_APPROVAL_RESPONSE_FAILED`
   - errorMessage: `result('Try')?[0]?['error']?['message']`
2. Respond failure (5 outputs).

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_ProcessApprovalResponse`
- [ ] Trigger: Power Apps (V2) with 6 inputs
- [ ] All 18 variables initialized; varCorrelationId auto-GUID
- [ ] Log Started before Try scope
- [ ] INVALID_OUTCOME validated before reading item
- [ ] Get item: 404 handled
- [ ] All 8 Svi predmeti fields read and stored in variables
- [ ] NOT_IN_APPROVAL validated (Stanje ≠ U odobravanju)
- [ ] APPROVAL_STATE_INVALID validated (KoraciJson empty or step not found)
- [ ] STEP_ALREADY_RESOLVED validated (step status ≠ Pending)
- [ ] User step: email comparison is case-insensitive
- [ ] Group step: Office365Groups member list checked; API failure is Warning (non-fatal)
- [ ] RESPONDER_NOT_AUTHORIZED returned on authorization failure
- [ ] Current step updated in KoraciJson (setProperty chain)
- [ ] Remaining steps marked Skipped on rejection
- [ ] History entry appended to IstorijaJson
- [ ] varNextAction set correctly in each branch
- [ ] Request digest obtained before PATCH
- [ ] PATCH uses If-Match with varDocETag
- [ ] 412 handling: re-read, check step status, retry once
- [ ] NextStepActivated: AssignPermissions called (non-fatal), email sent (non-fatal)
- [ ] ProcessComplete / DocumentRejected: initiator email sent (non-fatal)
- [ ] Log Success before Respond success
- [ ] Catch scope: Log Error, Respond failure
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow inside DocCentralV3 solution
