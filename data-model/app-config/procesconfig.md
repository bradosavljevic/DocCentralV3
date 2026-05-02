# AppConfig — ProcesConfig

## Svrha

Konfiguracija procesnih koraka odobravanja po tipu dokumenta, valuti, iznosu, redosledu i organizacionoj jedinici.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `ProcesConfig`
- SharePoint item ID: `20`
- Poslednja izmena: `12.8.2025. 12:26`
- Deklarisani `ColumnHeader`: ``
- Broj zapisa u JSON konfiguraciji: `6`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `TipDokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `Iznos` | Polje je pronađeno u JSON konfiguraciji. |
| `RedosledKoraka` | Polje je pronađeno u JSON konfiguraciji. |
| `RedosledKorakaTreshold` | Polje je pronađeno u JSON konfiguraciji. |
| `NacinOdobravanja` | Polje je pronađeno u JSON konfiguraciji. |
| `OdobravalacGrupa` | Polje je pronađeno u JSON konfiguraciji. |
| `NazivKoraka` | Polje je pronađeno u JSON konfiguraciji. |
| `Aktivan` | Polje je pronađeno u JSON konfiguraciji. |
| `KomentarObavezan` | Polje je pronađeno u JSON konfiguraciji. |
| `OrganizacionaJedinica` | Polje je pronađeno u JSON konfiguraciji. |
| `Napomena` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| TipDokumenta | Iznos | RedosledKoraka | RedosledKorakaTreshold | NacinOdobravanja | OdobravalacGrupa | NazivKoraka | Aktivan | KomentarObavezan | OrganizacionaJedinica | Napomena |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Fakture | [{"Valuta": "EUR", "Iznos": 100000}, {"Valuta": "USD", "Iznos": 500000}, {"Valuta": "USD", "Iznos": 500000}] | 1 | 1 | App | s_b.radosavljevic@gopro.rs | Odobrenje sektor menadžer | True | True |  | Za sve fakture manje od 100.000 RSD |
| Fakture | [{"Valuta": "EUR", "Iznos": 100000}, {"Valuta": "USD", "Iznos": 500000}, {"Valuta": "USD", "Iznos": 500000}] | 2 | 2 | App | s_b.radosavljevic@gopro.rs | Finansijska kontrola | True | True |  | Za sve fakture manje od 100.000 RSD |
| Fakture | [{"Valuta": "EUR", "Iznos": 100000}, {"Valuta": "USD", "Iznos": 500000}, {"Valuta": "USD", "Iznos": 500000}] | 3 | 3 | App | s_b.radosavljevic@gopro.rs | Finansijski menadzment kontrola | True | True |  | Za sve fakture manje od 100.000 RSD |
| Fakture | [{"Valuta": "EUR", "Iznos": 100000}, {"Valuta": "USD", "Iznos": 500000}, {"Valuta": "USD", "Iznos": 500000}] |  | 4 | App | s_b.radosavljevic@gopro.rs | Menadzment kontrola | True | True |  | Za sve fakture manje od 100.000 RSD |
| Fakture | [{"Valuta": "EUR", "Iznos": 100000}, {"Valuta": "USD", "Iznos": 500000}, {"Valuta": "USD", "Iznos": 500000}] | 4 | 5 | App | s_b.radosavljevic@gopro.rs | Administracija | True | True |  | Za sve fakture manje od 100.000 RSD |
| Fakture | [{"Valuta": "EUR", "Iznos": 1000}] | 1 | 1 | App | s_b.radosavljevic@gopro.rs | Odobrenje Accounting | True | True | Magacin | Za sve ugovore manje od 100.000 RSD |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `6` JSON zapisa.
- Pronađena polja: `TipDokumenta`, `Iznos`, `RedosledKoraka`, `RedosledKorakaTreshold`, `NacinOdobravanja`, `OdobravalacGrupa`, `NazivKoraka`, `Aktivan`, `KomentarObavezan`, `OrganizacionaJedinica`, `Napomena`.

## Nepoznato / potrebno potvrditi

- Tačan način na koji Canvas aplikacija i Power Automate koriste ovu konfiguraciju.
- Da li su svi procesni koraci aktivni u produkcionom toku.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
