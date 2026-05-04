# SharePoint lista: Rezervisani Brojevi

## Svrha

Lista služi za evidenciju unapred rezervisanih delovodnih brojeva.

## Pravila

- Kreiraju je korisnici koji zavode dokumente.
- Rezervisani broj može se izmeniti.
- Rezervisani broj se ne može ručno obrisati nakon rezervacije.
- Briše se automatski kada se iskoristi.
- Važi samo za jednu godinu.
- Može se koristiti za bilo koji tip dokumenta.
- Svi korisnici koji zavode dokumente vide sve rezervisane brojeve.
- Lista mora biti prazna pre zaključenja godine.

## Očekivana polja

- ID
- Title
- DelovodniBroj
- Godina
- DatumRezervacije
- DatumZavodjenja
- Rezervisao
- Napomena
- Created
- Modified
- Author
- Editor

## Implementacija

Korišćenje rezervisanog broja mora biti transakciono u okviru flow-a za zavođenje dokumenta.

Ako kreiranje dokumenta ne uspe, rezervisani broj se ne sme obrisati.
