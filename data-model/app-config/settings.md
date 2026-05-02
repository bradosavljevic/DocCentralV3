# AppConfig — Settings

## Svrha

Opšta podešavanja aplikacije, uključujući SharePoint link, procesni režim i admin naloge.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `Settings`
- SharePoint item ID: `19`
- Poslednja izmena: `10.6.2025. 13:21`
- Deklarisani `ColumnHeader`: ``
- Broj zapisa u JSON konfiguraciji: `1`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `SPLink` | Polje je pronađeno u JSON konfiguraciji. |
| `Proces` | Polje je pronađeno u JSON konfiguraciji. |
| `AdminNalog` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| SPLink | Proces | AdminNalog |
| --- | --- | --- |
| https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/Lists/SviPredmeti/AllItems.aspx | Ne | [{"Email": "bozidar@powerbml.rs"}, {"Email": "admin2@firma.com"}, {"Email": "admin3@firma.com"}] |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `1` JSON zapisa.
- Pronađena polja: `SPLink`, `Proces`, `AdminNalog`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
