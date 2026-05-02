# AppConfig — OrganizacioneJedinice

## Svrha

Šifarnik organizacionih jedinica i odgovornih menadžera/zamenika.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `OrganizacioneJedinice`
- SharePoint item ID: `5`
- Poslednja izmena: `23.4.2026. 14:02`
- Deklarisani `ColumnHeader`: `Title,MenadzerSektora,MenadzerSektoraEmail,ZamenikMenadzerSektora,ZamenikMenadzerSektoraEmail`
- Broj zapisa u JSON konfiguraciji: `15`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `MenadzerSektora` | Polje je pronađeno u JSON konfiguraciji. |
| `MenadzerSektoraEmail` | Polje je pronađeno u JSON konfiguraciji. |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `ZamenikMenadzerSektora` | Polje je pronađeno u JSON konfiguraciji. |
| `ZamenikMenadzerSektoraEmail` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| MenadzerSektora | MenadzerSektoraEmail | Title | ZamenikMenadzerSektora | ZamenikMenadzerSektoraEmail |
| --- | --- | --- | --- | --- |
| Gordana Stefanovic | gordana.stefanovic@gopro.rs | IT |  |  |
| Gordana Stefanovic | gordana.stefanovic@gopro.rs | EHS |  |  |
| Gordana Stefanovic | gordana.stefanovic@gopro.rs | Finansije |  |  |
| Gordana Stefanovic | gordana.stefanovic@gopro.rs | HR |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Kancelarija direktora |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Inzenjering |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Odrzavanje |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Logistika |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Kvalitet |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Proizvodnja |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Nabavka |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Asistent direktora |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Financial Manager |  |  |
| Božidar Radosavljević | s_b.radosavljevic@gopro.rs | Top manadzment |  |  |
| Božidar Radosavljević | gordana.stefanovic@gopro.rs | Administration |  |  |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `15` JSON zapisa.
- Pronađena polja: `MenadzerSektora`, `MenadzerSektoraEmail`, `Title`, `ZamenikMenadzerSektora`, `ZamenikMenadzerSektoraEmail`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
