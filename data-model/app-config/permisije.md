# AppConfig — Permisije

## Svrha

Konfiguracija pravila pristupa po kategoriji, tipu dokumenta, ulazno/izlazno, organizacionoj jedinici i grupi.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `Permisije`
- SharePoint item ID: `8`
- Poslednja izmena: `24.6.2025. 10:44`
- Deklarisani `ColumnHeader`: `Title,DocumentCategory,InputOutput,DocumentType,OrganizationalUnit,Group,GroupEmail`
- Broj zapisa u JSON konfiguraciji: `2`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `DocumentCategory` | Polje je pronađeno u JSON konfiguraciji. |
| `DocumentType` | Polje je pronađeno u JSON konfiguraciji. |
| `Group` | Polje je pronađeno u JSON konfiguraciji. |
| `GroupEmail` | Polje je pronađeno u JSON konfiguraciji. |
| `InputOutput` | Polje je pronađeno u JSON konfiguraciji. |
| `OrganizationalUnit` | Polje je pronađeno u JSON konfiguraciji. |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| DocumentCategory | DocumentType | Group | GroupEmail | InputOutput | OrganizationalUnit | Title |
| --- | --- | --- | --- | --- | --- | --- |
| Finansije | Avansni račun | Lazar Magovčević | test@test.com | Ulazni | Administration | 3ac780fb-987b-442b-b783-ccdd5abcad2a |
| Administrativna dokumenta | Dopisi, Obaveštenja, Podnesci | Zoli Playground Members | test@test.com | Ulazni | Administration | 56c50b70-76d0-494d-bc16-9c5da7e58746 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `2` JSON zapisa.
- Pronađena polja: `DocumentCategory`, `DocumentType`, `Group`, `GroupEmail`, `InputOutput`, `OrganizationalUnit`, `Title`.

## Nepoznato / potrebno potvrditi

- Da li ova konfiguracija upravlja stvarnim SharePoint permisijama ili samo aplikativnim prikazom.
- Da li se grupe mapiraju na Entra ID / Microsoft 365 grupe.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
