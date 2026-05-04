# Field mapping: CF_DocCentralV3_ProcessApprovalResponse

## Svi predmeti — fields read

| Display name | Internal name | Expression | Notes |
|---|---|---|---|
| Stanje | UNKNOWN | `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN']` | Pre-condition check |
| InicijatorEmail | UNKNOWN | same | Used to notify on rejection/completion |
| TrenutniOdobravalacEmail | UNKNOWN | same | Authorization check (user step) |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | same | Authorization check (group step) |
| TrenutniKorakOdobravanja | UNKNOWN | same | Current step number |
| UkupnoKorakaOdobravanja | UNKNOWN | same | Total steps — used to detect final step |
| KoraciOdobravanjaJson | UNKNOWN | same | Step array to update |
| IstorijaOdobravanjaJson | UNKNOWN | same | History array to append to |
| DelovodniBroj | UNKNOWN | same | Used in notification email subjects and log messages |
| @odata.etag | system | `outputs('Get_SviPredmeti_Item')?['body']?['@odata.etag']` | Used in If-Match header |

---

## Svi predmeti — fields written (HTTP PATCH)

Three different PATCH bodies depending on outcome branch. All internal names UNKNOWN.

### Branch A: Rejection

| Display name | Internal name | Value |
|---|---|---|
| Stanje | UNKNOWN | `'Odbijeno'` |
| TrenutniOdobravalacEmail | UNKNOWN | `''` |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | `''` |
| TrenutniKorakOdobravanja | UNKNOWN | `0` (or leave unchanged — confirm with client) |
| KoraciOdobravanjaJson | UNKNOWN | varUpdatedKoraciJson (current step Rejected, remaining Skipped) |
| IstorijaOdobravanjaJson | UNKNOWN | varUpdatedIstorijaJson (new entry appended) |

### Branch B: Approved, next step exists

| Display name | Internal name | Value |
|---|---|---|
| Stanje | UNKNOWN | unchanged (`U odobravanju`) |
| TrenutniKorakOdobravanja | UNKNOWN | `varTrenutniKorak + 1` |
| TrenutniOdobravalacEmail | UNKNOWN | `varNaredniKorak?['assigneeEmail']` or `''` |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | `varNaredniKorak?['assigneeGroupId']` or `''` |
| KoraciOdobravanjaJson | UNKNOWN | varUpdatedKoraciJson (current step Approved) |
| IstorijaOdobravanjaJson | UNKNOWN | varUpdatedIstorijaJson |

### Branch C: Approved, no next step (ProcessComplete)

| Display name | Internal name | Value |
|---|---|---|
| Stanje | UNKNOWN | `'Odobreno'` |
| TrenutniOdobravalacEmail | UNKNOWN | `''` |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | `''` |
| TrenutniKorakOdobravanja | UNKNOWN | `0` (or leave unchanged) |
| KoraciOdobravanjaJson | UNKNOWN | varUpdatedKoraciJson (all steps Approved) |
| IstorijaOdobravanjaJson | UNKNOWN | varUpdatedIstorijaJson |

---

## JSON update operations

### Update current step in KoraciOdobravanjaJson

See `PACode/flows/approval-runtime-json-pattern.md` — section "Pattern: Update one element in an array".

Key expression to find and update the current step:
```
[Target step] = first(filter(json(variables('varKoraciJson')), equals(item()?['stepNumber'], variables('varTrenutniKorak'))))
[Updated step] = setProperty(setProperty(setProperty(setProperty(<targetStep>, 'status', triggerBody()?['text_outcome']), 'resolvedBy', triggerBody()?['text_responderEmail']), 'resolvedAt', utcNow()), 'comments', triggerBody()?['text_comments'])
[Other steps] = filter(json(variables('varKoraciJson')), not(equals(item()?['stepNumber'], variables('varTrenutniKorak'))))
[varUpdatedKoraciJson] = string(concat(filter(json(variables('varKoraciJson')), less(item()?['stepNumber'], variables('varTrenutniKorak'))), array(<updatedStep>), filter(json(variables('varKoraciJson')), greater(item()?['stepNumber'], variables('varTrenutniKorak')))))
```

This preserves step order: steps before + updated step + steps after.

### Mark remaining steps Skipped (rejection only)

After updating the current step to Rejected, update all steps where `stepNumber > varTrenutniKorak`:
```
varUpdatedKoraciJson = string(
  concat(
    filter(json(varUpdatedKoraciJson), lessOrEquals(item()?['stepNumber'], variables('varTrenutniKorak'))),
    map(filter(json(varUpdatedKoraciJson), greater(item()?['stepNumber'], variables('varTrenutniKorak'))), setProperty(item(), 'status', 'Skipped'))
  )
)
```

`map()` is available in Power Automate expressions. Test this in the expression editor first.
If `map()` is not available, use Apply to each with Append to array variable instead.

### Append to IstorijaOdobravanjaJson

See `PACode/flows/approval-runtime-json-pattern.md` — section "Pattern: Append to history array".

The history entry must include the current round number. Compute from IstorijaJson:
```
varCurrentRound = add(length(filter(json(variables('varIstorijaJson')), equals(item()?['outcome'], 'RoundReset'))), 1)
```

---

## Notification email targets

| Scenario | Recipient | Subject |
|---|---|---|
| Next step activated | Next step's assignee(s) | `concat('DocCentral: Zahtev za odobrenje dokumenta ', varDelovodniBroj)` |
| Process complete | InicijatorEmail | `concat('DocCentral: Dokument odobren - ', varDelovodniBroj)` |
| Document rejected | InicijatorEmail | `concat('DocCentral: Dokument odbijen - ', varDelovodniBroj)` |

Email body templates: UNKNOWN — confirm with client.

---

## Actions to complete before building

1. Read all Svi predmeti internal column names.
2. Fill in all UNKNOWN entries above.
3. Confirm email body templates.
4. Confirm TrenutniKorakOdobravanja value on rejection/completion (0 or leave unchanged).
5. Confirm permission retention policy for previous approvers (read App Config UNKNOWN key).
