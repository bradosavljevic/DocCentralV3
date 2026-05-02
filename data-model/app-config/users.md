# AppConfig — Users

## Svrha

Korisničke postavke, uključujući jezik i rolu gde postoji.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `Users`
- SharePoint item ID: `14`
- Poslednja izmena: `21.4.2026. 11:47`
- Deklarisani `ColumnHeader`: `Title,User,Lang`
- Broj zapisa u JSON konfiguraciji: `3`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Lang` | Polje je pronađeno u JSON konfiguraciji. |
| `Rola` | Polje je pronađeno u JSON konfiguraciji. |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `User` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Lang | Rola | Title | User |
| --- | --- | --- | --- |
| rs |  | b77b9d4c-4752-4c36-9c5a-d6a286ffbd03 | s_b.radosavljevic@gopro.rs |
| rs |  | 59519e1d-4266-41de-b293-09195a07300a | gordana.stefanovic@gopro.rs |
| en |  | dc9c3e7c-1ea7-4da5-81c1-60deeac8dc94 | kristina.lecic@gopro.rs |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `3` JSON zapisa.
- Pronađena polja: `Lang`, `Rola`, `Title`, `User`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
