# AppConfig — ArhivskaKnjigaGrupeKategorija

## Svrha

Grupisanje arhivskih kategorija dokumentarnog materijala po rimskim rednim oznakama.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `ArhivskaKnjigaGrupeKategorija`
- SharePoint item ID: `1`
- Poslednja izmena: `5.8.2025. 10:30`
- Deklarisani `ColumnHeader`: `Title,RedniBroj`
- Broj zapisa u JSON konfiguraciji: `11`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `RedniBroj` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | RedniBroj |
| --- | --- |
| Predmeti koji se odnose na osnivanje i organizaciju Društva | I |
| Pravni i opšti poslovi | II |
| Pravilnici i drugi opšti akti | III |
| Predmeti iz oblasti radnih odnosa | IV |
| Predmeti koji se odnose na investicije, izgradnju i adaptaciju objekata | V |
| Kancelarijsko i arhivsko poslovanje | VI |
| Dokumentacija osnovne delatnosti | VII |
| Finansijsko-materijalno poslovanje | VIII |
| Dokumentacija informacionih sistema (IS) | IX |
| Evidencija iz oblasti bezbednosti i zaštite na radu (BZR), zaštite od požara (ZOP) i zaštite životne sredine (ZŽS) | X |
| Dokumentacija sistema kvaliteta prema zahtevima standarda SRPS ISO 9000 (ako privredno društvo poseduje sertifikat) | XI |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `11` JSON zapisa.
- Pronađena polja: `Title`, `RedniBroj`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
