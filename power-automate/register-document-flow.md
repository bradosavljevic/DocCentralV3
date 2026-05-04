# Flow: RegisterDocument

## Tip

Power Apps triggered flow.

## Svrha

Centralni flow za zavođenje novog dokumenta.

## Input

Očekivani input:

- tip dokumenta
- metadata obavezna polja
- prilog/fajl informacije
- izbor režima broja:
  - reserved
  - next
- reserved number id ako se koristi
- datum zavođenja
- korisnik/inicijator
- organizaciona jedinica
- partner
- dodatni primaoci / prava

## Koraci

1. Kreiraj CorrelationId.
2. Validiraj input.
3. Proveri aktivnu godinu.
4. Proveri da godina nije zaključana.
5. Ako se koristi rezervisani broj:
   - validiraj broj
   - proveri godinu
   - koristi broj
6. Ako se koristi sledeći broj:
   - generiši broj kroz concurrency-safe mehanizam
   - uvećaj brojač
7. Kreiraj SharePoint item u Svi predmeti.
8. Kreiraj folder/biblioteku za dokument.
9. Dodeli prava.
10. Ako je korišćen rezervisani broj, obriši ga iz liste.
11. Loguj uspeh.
12. Vrati response aplikaciji.

## Error handling

U slučaju greške:

- logovati grešku
- vratiti korisniku kontrolisanu poruku
- ne brisati rezervisani broj ako dokument nije uspešno kreiran
- ne povećavati brojač ako dokument nije uspešno zaveden
- rollback gde je moguće

## Output

Primer output-a:

```json
{
  "success": true,
  "documentId": 123,
  "delovodniBroj": "2026-0001",
  "message": "Dokument je uspešno zaveden."
}
```

Greška:

```json
{
  "success": false,
  "errorCode": "REGISTRY_NUMBER_CONFLICT",
  "message": "Delovodni broj nije mogao biti generisan. Pokušajte ponovo."
}
```
