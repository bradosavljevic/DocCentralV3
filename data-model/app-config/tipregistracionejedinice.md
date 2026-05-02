# AppConfig — TipRegistracioneJedinice

## Svrha

Šifarnik tipova registracionih jedinica.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `TipRegistracioneJedinice`
- SharePoint item ID: `12`
- Poslednja izmena: `5.8.2025. 10:32`
- Deklarisani `ColumnHeader`: `Title`
- Broj zapisa u JSON konfiguraciji: `3`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title |
| --- |
| Registrator |
| Fascikla |
| Kutija |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `3` JSON zapisa.
- Pronađena polja: `Title`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
