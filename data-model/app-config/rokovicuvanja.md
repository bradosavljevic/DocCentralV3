# AppConfig — RokoviCuvanja

## Svrha

Šifarnik rokova čuvanja dokumentacije izražen kroz naziv, broj meseci i oznaku trajnog čuvanja.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `RokoviCuvanja`
- SharePoint item ID: `9`
- Poslednja izmena: `29.5.2025. 10:20`
- Deklarisani `ColumnHeader`: `Titl,BrojMeseci,TrajnoCuvanje`
- Broj zapisa u JSON konfiguraciji: `12`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `BrojMeseci` | Polje je pronađeno u JSON konfiguraciji. |
| `TrajnoCuvanje` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | BrojMeseci | TrajnoCuvanje |
| --- | --- | --- |
| Trajno | 0 | true |
| 5 godina | 60 | false |
| 10 godina | 120 | false |
| 10 godina od isteka zakupa | 120 | false |
| 10 godina (po okončanju predmeta) | 120 | false |
| 5 godina po okončanju predmeta | 60 | false |
| 3 godine | 36 | false |
| 2 godine | 24 | false |
| 20 godina | 24 | false |
| 40 godina | 48 | false |
| 6 godina od dana prestanka važenja stručnog nalaza | 72 | false |
| 3 godine od dana prestanka korišćenja opasne materije | 36 | false |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `12` JSON zapisa.
- Pronađena polja: `Title`, `BrojMeseci`, `TrajnoCuvanje`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
