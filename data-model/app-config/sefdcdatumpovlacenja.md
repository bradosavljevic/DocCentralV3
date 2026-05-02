# AppConfig — SEFDCDatumPovlacenja

## Svrha

Tehnička konfiguracija poslednjeg datuma/vremena povlačenja SEF podataka u Document Central.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `SEFDCDatumPovlacenja`
- SharePoint item ID: `24`
- Poslednja izmena: `21.4.2026. 16:02`
- Deklarisani `ColumnHeader`: ``
- Broj zapisa u JSON konfiguraciji: `1`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `LastDate` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| LastDate |
| --- |
| 2026-04-21 14:00:00 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `1` JSON zapisa.
- Pronađena polja: `LastDate`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
