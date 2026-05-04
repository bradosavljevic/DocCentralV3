# Process: Unos novog dokumenta

## Tok procesa

1. Korisnik otvara aplikaciju.
2. Korisnik bira tip dokumenta.
3. Korisnik popunjava obavezna polja.
4. Korisnik dodaje prilog.
5. Korisnik bira da li koristi rezervisani broj ili generiše sledeći broj u nizu.
6. Ako koristi rezervisani broj, bira datum zavođenja.
7. Ako koristi sledeći broj u nizu, datum zavođenja se automatski postavlja na današnji datum.
8. Korisnik klikće `Zavedi`.
9. Sistem proverava validaciju.
10. Sistem preuzima ili generiše delovodni broj.
11. Ako broj nije rezervisan, sistem povećava counter za sledeće zavođenje.
12. Ako je broj rezervisan, sistem ga nakon uspešnog zavođenja briše iz liste rezervisanih brojeva.
13. Sistem kreira SharePoint item.
14. Sistem kreira folder/biblioteku za dokument.
15. Sistem dodeljuje prava.
16. Sistem prikazuje poruku korisniku.

## Pravila

- delovodni broj mora biti jedinstven
- korisnik ne sme dobiti isti broj kao drugi korisnik
- zavođenje mora biti atomic koliko god SharePoint / Power Automate model dozvoljava
- neuspešno kreiranje dokumenta mora biti logovano
- neuspešan pokušaj generisanja broja mora biti logovan

## UI napomena

`Novi predmet` je forma za unos, ne ekran za pregled svih predmeta.
