# Proces: Unos novog dokumenta

## Cilj

Omogućiti korisniku da zavede novi dokument kroz Canvas aplikaciju, uz validaciju, dodelu jedinstvenog delovodnog broja, kreiranje SharePoint item-a, kreiranje foldera/biblioteke, dodelu prava i korisničku potvrdu.

## Tok procesa

1. Korisnik otvara aplikaciju.
2. Korisnik bira tip dokumenta.
3. Korisnik popunjava obavezna polja.
4. Korisnik dodaje prilog.
5. Korisnik bira način dodele delovodnog broja:
   - koristi rezervisani broj
   - generiše sledeći broj u nizu
6. Ako korisnik koristi rezervisani broj:
   - bira datum zavođenja
   - bira postojeći rezervisani broj
7. Ako korisnik koristi sledeći broj u nizu:
   - datum zavođenja se automatski postavlja na današnji datum
8. Korisnik klikće `Zavedi`.
9. Sistem radi validaciju.
10. Sistem preuzima ili generiše delovodni broj.
11. Sistem kreira SharePoint item.
12. Sistem kreira folder/biblioteku za dokument.
13. Sistem dodeljuje prava.
14. Sistem prikazuje poruku korisniku.

## Pravila za delovodni broj

Ako broj nije rezervisan:

- sistem generiše sledeći broj u nizu
- sistem ga upisuje u dokument
- sistem povećava brojač za sledeće zavođenje

Ako je broj rezervisan:

- sistem koristi izabrani rezervisani broj
- nakon uspešnog zavođenja sistem briše rezervisani broj iz liste rezervisanih brojeva

## Validacija

Minimalne validacije:

- tip dokumenta mora biti izabran
- obavezna polja moraju biti popunjena
- prilog mora biti dodat
- delovodni broj mora biti dostupan
- rezervisani broj mora pripadati aktivnoj godini
- zaključana godina ne sme dozvoliti novo zavođenje
- korisnik mora imati pravo za zavođenje dokumenta

## Tehnički zahtev

Create operacija mora ići preko Power Automate flow-a pod servisnim nalogom.

Canvas app ne sme direktno kreirati dokumente ako korisnici imaju samo Read Only prava nad SharePoint-om.

## Kritičan enterprise zahtev

Generisanje delovodnog broja mora biti concurrency-safe.

Sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
