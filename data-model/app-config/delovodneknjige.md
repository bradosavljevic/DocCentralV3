# AppConfig — DelovodneKnjige

## Svrha

Konfiguracija delovodnika, prefiksa i poslednjeg korišćenog delovodnog broja po godini.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `DelovodneKnjige`
- SharePoint item ID: `4`
- Poslednja izmena: `29.4.2026. 09:01`
- Deklarisani `ColumnHeader`: `InkrementUvecanja,PoslednjiDelovodniBroj,PrefiksZaDelovodniBroj,Title`
- Broj zapisa u JSON konfiguraciji: `2`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `PoslednjiDelovodniBroj` | Polje je pronađeno u JSON konfiguraciji. |
| `PrefiksZaDelovodniBroj` | Polje je pronađeno u JSON konfiguraciji. |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `Godina` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| PoslednjiDelovodniBroj | PrefiksZaDelovodniBroj | Title | Godina |
| --- | --- | --- | --- |
| 54 | Os.Del.Br. | Osnovni delovodnik | 2026 |
| 1 | BDB | Bez delovodnog broja | 2026 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `2` JSON zapisa.
- Pronađena polja: `PoslednjiDelovodniBroj`, `PrefiksZaDelovodniBroj`, `Title`, `Godina`.

## Nepoznato / potrebno potvrditi

- Da li Power Automate koristi ovu konfiguraciju kao jedini izvor istine u trenutku upisa.
- Da li postoji atomic/optimistic locking i retry logika.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
