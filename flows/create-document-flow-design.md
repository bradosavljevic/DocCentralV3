# Flow design: CF_DocCentralV3_CreateDocument

## Namena

Centralni flow za zavođenje dokumenta.

## Trigger

Power Apps V2 trigger.

## Input

Preporučeni input:

```json
{
  "documentType": "",
  "metadata": {},
  "useReservedNumber": false,
  "reservedNumberId": null,
  "filingDate": "",
  "mainFile": {},
  "attachments": [],
  "initiatorEmail": "",
  "correlationId": ""
}
```

## Koraci

1. Inicijalizuj `CorrelationId`.
2. Validiraj input.
3. Proveri tip dokumenta.
4. Proveri obavezna polja.
5. Proveri da postoji glavni dokument ili očekivani fajl.
6. Ako korisnik koristi rezervisani broj, pozovi/izvrši logiku `UseReservedNumber`.
7. Ako korisnik generiše novi broj, pozovi/izvrši logiku `GenerateRegistryNumber`.
8. Kreiraj item u `Svi predmeti`.
9. Generiši unikatan naziv glavnog fajla.
10. Kreiraj glavni fajl u root-u biblioteke.
11. Upisi metapodatke na glavni fajl.
12. Za svaki prilog generiši unikatan naziv fajla.
13. Kreiraj prilog u root-u biblioteke.
14. Upisi metapodatke na prilog.
15. Dodeli prava na item i fajlove.
16. Ako je korišćen novi broj, povećaj brojač.
17. Ako je korišćen rezervisani broj, ukloni ga iz liste.
18. Upisi audit log.
19. Vrati response aplikaciji.

## Anti-overwrite

Za svaki fajl koristiti:

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

Za prilog:

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

## Error handling

Koristiti scopes:

- Try
- Catch
- Finally

U Catch obavezno pozvati:

```text
CF_DocCentralV3_LogEvent
```

## Response

Uspešan response:

```json
{
  "success": true,
  "message": "Dokument je uspešno zaveden.",
  "itemId": 0,
  "delovodniBroj": "",
  "correlationId": ""
}
```

Neuspešan response:

```json
{
  "success": false,
  "message": "Dokument nije zaveden.",
  "errorCode": "",
  "correlationId": ""
}
```
