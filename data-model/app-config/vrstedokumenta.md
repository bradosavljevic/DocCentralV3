# AppConfig — VrsteDokumenta

## Svrha

Šifarnik glavnih vrsta dokumenata.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `VrsteDokumenta`
- SharePoint item ID: `16`
- Poslednja izmena: `27.9.2025. 11:50`
- Deklarisani `ColumnHeader`: `Title`
- Broj zapisa u JSON konfiguraciji: `6`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title |
| --- |
| Ugovori |
| Statusna dokumenta |
| Administrativna dokumenta |
| Radni odnosi |
| Finansije |
| Ostalo |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `6` JSON zapisa.
- Pronađena polja: `Title`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
