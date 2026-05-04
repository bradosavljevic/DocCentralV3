# Field mapping: CF_DocCentralV3_SendForApproval

## Svi predmeti — fields read

| Display name | Internal name | How read | Notes |
|---|---|---|---|
| Stanje | UNKNOWN | `outputs('Get_SviPredmeti_Item')?['body']?['UNKNOWN']` | Pre-condition check |
| IstorijaOdobravanjaJson | UNKNOWN | Same | Preserved / extended on resubmission |
| @odata.etag | system | `outputs('Get_SviPredmeti_Item')?['body']?['@odata.etag']` | Used in If-Match header |
| ID | ID | `outputs('Get_SviPredmeti_Item')?['body']?['ID']` | Confirmation |

## Svi predmeti — fields written (via HTTP PATCH)

All display names confirmed. Internal names UNKNOWN — replace before building.

| Display name | Internal name | Value written | Type |
|---|---|---|---|
| Stanje | UNKNOWN | `'U odobravanju'` | Choice/Text |
| TrenutniKorakOdobravanja | UNKNOWN | `1` | Number |
| TrenutniOdobravalacEmail | UNKNOWN | step1.assigneeEmail or `''` | Single line of text |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | step1.assigneeGroupId or `''` | Single line of text |
| UkupnoKorakaOdobravanja | UNKNOWN | `length(varApprovalSteps)` | Number |
| KoraciOdobravanjaJson | UNKNOWN | `varKoraciJson` (full step array JSON string) | Multiple lines of text |
| IstorijaOdobravanjaJson | UNKNOWN | `varIstorijaJson` (`'[]'` or extended) | Multiple lines of text |

## App Config — fields read for ProcesConfig

All UNKNOWN until AppConfig.csv is confirmed.

| What is needed | App Config key | Internal column | Notes |
|---|---|---|---|
| ProcesConfig for documentType | UNKNOWN | UNKNOWN | JSON string with steps array |
| Default/fallback ProcesConfig | UNKNOWN | UNKNOWN | Used when no type-specific config |

## PATCH request structure (HTTP)

```
Method: POST (with X-HTTP-Method: MERGE)
Uri: _api/web/lists/getbytitle('<SviPredmetiList>')/items(<documentItemId>)
Headers:
  Accept: application/json;odata=nometadata
  Content-Type: application/json;odata=nometadata
  IF-MATCH: <varDocETag>
  X-HTTP-Method: MERGE
Body:
{
  "UNKNOWN_Stanje": "U odobravanju",
  "UNKNOWN_TrenutniKorakOdobravanja": 1,
  "UNKNOWN_TrenutniOdobravalacEmail": "<step1Email>",
  "UNKNOWN_TrenutnaGrupaOdobravanjaId": "<step1GroupId>",
  "UNKNOWN_UkupnoKorakaOdobravanja": <stepCount>,
  "UNKNOWN_KoraciOdobravanjaJson": "<varKoraciJson>",
  "UNKNOWN_IstorijaOdobravanjaJson": "<varIstorijaJson>"
}
```

Replace all UNKNOWN_ prefixes with actual internal names once list schema is confirmed.

## Notification email fields

| Field | Source | Notes |
|---|---|---|
| To | step 1 assigneeEmail (User) or all group members (Group) | Resolved per step type |
| Subject | `concat('DocCentral: Zahtev za odobrenje dokumenta ', delovodniBroj)` | |
| Body | UNKNOWN template | Placeholder: delovodniBroj, initiator name, instruction to open Canvas App |

## Actions to complete before building

1. Read Svi predmeti column internal names from List Settings → update all UNKNOWN fields.
2. Read App Config ProcesConfig key from AppConfig.csv or list.
3. Confirm email body template with client.
4. Confirm whether InicijatorEmail must be (re)written by this flow or only by CreateDocument.
