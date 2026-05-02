# AppConfig — RegistracioneJedinice

## Svrha

Evidencija registracionih jedinica po lokaciji, tipu, godini, klasifikacionoj oznaci i AK kategoriji.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `RegistracioneJedinice`
- SharePoint item ID: `17`
- Poslednja izmena: `27.2.2026. 10:05`
- Deklarisani `ColumnHeader`: `Title,Lokacija,TipRegistracioneJedinice,GodinaNastanka,KlasifikacionaOznaka,AKKategorija,AKKategorijaGrupaKategorija`
- Broj zapisa u JSON konfiguraciji: `6`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `Lokacija` | Polje je pronađeno u JSON konfiguraciji. |
| `TipRegistracioneJedinice` | Polje je pronađeno u JSON konfiguraciji. |
| `BrojRegistracioneJedinice` | Polje je pronađeno u JSON konfiguraciji. |
| `GodinaNastanka` | Polje je pronađeno u JSON konfiguraciji. |
| `KlasifikacionaOznaka` | Polje je pronađeno u JSON konfiguraciji. |
| `AKKategorija` | Polje je pronađeno u JSON konfiguraciji. |
| `AKKategorijaGrupaKategorija` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | Lokacija | TipRegistracioneJedinice | BrojRegistracioneJedinice | GodinaNastanka | KlasifikacionaOznaka | AKKategorija | AKKategorijaGrupaKategorija |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 4cde1123-51e0-4e94-adb9-8c1c611b5651 | Prostorija 1 | Registrator | 1 | 2025 | IX-216 | Dokumentacija sistemskog i aplikativnog softvera | Dokumentacija informacionih sistema (IS) |
| fdcb525b-d2e3-445c-bf03-c487b4e48233 | Prostorija 1 | Registrator | 2 | 2025 | IX-216 | Dokumentacija sistemskog i aplikativnog softvera | Dokumentacija informacionih sistema (IS) |
| 320bdd21-ef51-48d6-b602-6b2cb2354396 | Prostorija 2 | Registrator | 1 | 2025 | VIII-96 | Početni bilansi | Finansijsko-materijalno poslovanje |
| 2adb74ec-1d9b-454d-8fda-4d45643e0c3c | Prostorija 1 | Registrator | 1 | 2025 | IX-218 | Godišnji programi i izveštaji | Dokumentacija informacionih sistema (IS) |
| 5f031073-915b-4004-8e05-bfffc53c8406 | Prostorija 1 | Registrator | 1 | 2025 | II-31 | Parnični predmeti | Pravni i opšti poslovi |
| b20841f0-4bda-4d61-8e7e-e49e45dbe23b | Prostorija 1 | Registrator | 1 | 2026 | IX-216 | Dokumentacija sistemskog i aplikativnog softvera | Dokumentacija informacionih sistema (IS) |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `6` JSON zapisa.
- Pronađena polja: `Title`, `Lokacija`, `TipRegistracioneJedinice`, `BrojRegistracioneJedinice`, `GodinaNastanka`, `KlasifikacionaOznaka`, `AKKategorija`, `AKKategorijaGrupaKategorija`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
