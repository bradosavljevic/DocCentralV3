# AppConfig — Stanja

## Svrha

Konfiguracija statusa dokumenta/procesa, početnih i krajnjih stanja, arhiviranja i promene stanja.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `Stanja`
- SharePoint item ID: `10`
- Poslednja izmena: `24.6.2025. 15:31`
- Deklarisani `ColumnHeader`: `Title,TipDokumenta,DozvoliArhiviranje,OmoguciPromenuStanja,PocetnoStanje,KrajnjeStanje,IsProces,ApprovalType,Person,PersonEmail,Treshold,RedolsedStanjaTreshold,RedolsedStanja`
- Broj zapisa u JSON konfiguraciji: `11`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `TipDokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `DozvoliArhiviranje` | Polje je pronađeno u JSON konfiguraciji. |
| `OmoguciPromenuStanja` | Polje je pronađeno u JSON konfiguraciji. |
| `PocetnoStanje` | Polje je pronađeno u JSON konfiguraciji. |
| `KrajnjeStanje` | Polje je pronađeno u JSON konfiguraciji. |
| `IsProces` | Polje je pronađeno u JSON konfiguraciji. |
| `ApprovalType` | Polje je pronađeno u JSON konfiguraciji. |
| `Person` | Polje je pronađeno u JSON konfiguraciji. |
| `PearsonEmail` | Polje je pronađeno u JSON konfiguraciji. |
| `Treshold` | Polje je pronađeno u JSON konfiguraciji. |
| `RedolsedStanjaTreshold` | Polje je pronađeno u JSON konfiguraciji. |
| `RedosledStanja` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | TipDokumenta | DozvoliArhiviranje | OmoguciPromenuStanja | PocetnoStanje | KrajnjeStanje | IsProces | ApprovalType | Person | PearsonEmail | Treshold | RedolsedStanjaTreshold | RedosledStanja |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Zavedeno | Default | True | True | True | False | False |  |  |  | False |  | 1 |
| Arhivirano | Default |  | False | False | True | False |  |  |  | False |  | 2 |
| Provera fakture MS | Fakture |  | True | True | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 1 | 1 |
| Odobrenje GM | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 2 |  |
| Odobrenje FM | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 3 | 2 |
| Odobrenje Finansije | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 4 | 3 |
| Knjizenje NAV | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 5 | 4 |
| Zavedeno | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 5 | 6 |
| Arhivirano | Fakture |  | False | False | True | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 0 | 0 |
| Otkazano | Fakture |  | True | False | True | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 0 | 0 |
| Dopuna | Fakture |  | True | False | False | True | app | Bozidar Radosavljevic | bozidar@powerbml.rs | True | 0 | 0 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `11` JSON zapisa.
- Pronađena polja: `Title`, `TipDokumenta`, `DozvoliArhiviranje`, `OmoguciPromenuStanja`, `PocetnoStanje`, `KrajnjeStanje`, `IsProces`, `ApprovalType`, `Person`, `PearsonEmail`, `Treshold`, `RedolsedStanjaTreshold`, `RedosledStanja`.

## Nepoznato / potrebno potvrditi

- Tačan način na koji Canvas aplikacija i Power Automate koriste ovu konfiguraciju.
- Da li su svi procesni koraci aktivni u produkcionom toku.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
