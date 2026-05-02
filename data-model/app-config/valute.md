# AppConfig — Valute

## Svrha

Šifarnik valuta i pragova za procesna pravila.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `Valute`
- SharePoint item ID: `15`
- Poslednja izmena: `22.7.2025. 07:35`
- Deklarisani `ColumnHeader`: `Title,Treshold`
- Broj zapisa u JSON konfiguraciji: `10`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `Treshold` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | Treshold |
| --- | --- |
| RSD | 150,000.00 |
| USD | 0.00 |
| EUR | 1,500.00 |
| SEK | 0.00 |
| PLN | 0.00 |
| CHF | 0.00 |
| JPN | 0.00 |
| GBP | 0.00 |
| JPY  | 0.00 |
| INR | 0.00 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `10` JSON zapisa.
- Pronađena polja: `Title`, `Treshold`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
