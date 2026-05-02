# AppConfig — EfaktureParametri

## Svrha

Alternativna/lokalizovana konfiguracija parametara e-faktura sa nazivima polja na srpskom.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `EfaktureParametri`
- SharePoint item ID: `21`
- Poslednja izmena: `4.8.2025. 14:04`
- Deklarisani `ColumnHeader`: ``
- Broj zapisa u JSON konfiguraciji: `4`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Naziv dokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `Šifra dokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `Vrsta dokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `Tip dokumenta` | Polje je pronađeno u JSON konfiguraciji. |
| `Delovodnik` | Polje je pronađeno u JSON konfiguraciji. |
| `AK grupa kategorije` | Polje je pronađeno u JSON konfiguraciji. |
| `AK Kategorija` | Polje je pronađeno u JSON konfiguraciji. |
| `AK Klasifikaciona oznaka` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Naziv dokumenta | Šifra dokumenta | Vrsta dokumenta | Tip dokumenta | Delovodnik | AK grupa kategorije | AK Kategorija | AK Klasifikaciona oznaka |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Fakture | 380 | Finansije | Fakture | Osnovni delovodnik | Finansijsko-materijalno poslovanje | Knjiga ulaznih računa | VIII-110 |
| Avansni račun | 386 | Finansije | Avansni račun | Osnovni delovodnik | Finansijsko-materijalno poslovanje | Avansi | VIII-184 |
| Knjižno odobrenje | 381 | Finansije | Knjižno odobrenje | Osnovni delovodnik | Finansijsko-materijalno poslovanje | Knjižna pisma | VIII-183 |
| Knjižno zaduženje | 383 | Finansije | Knjižno zaduženje | Osnovni delovodnik | Finansijsko-materijalno poslovanje | Knjižna pisma | VIII-183 |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `4` JSON zapisa.
- Pronađena polja: `Naziv dokumenta`, `Šifra dokumenta`, `Vrsta dokumenta`, `Tip dokumenta`, `Delovodnik`, `AK grupa kategorije`, `AK Kategorija`, `AK Klasifikaciona oznaka`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
