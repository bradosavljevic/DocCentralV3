# AppConfig — TipoviDokumenta

## Svrha

Šifarnik tipova dokumenata vezanih za višu vrstu dokumenta.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `TipoviDokumenta`
- SharePoint item ID: `11`
- Poslednja izmena: `27.9.2025. 11:53`
- Deklarisani `ColumnHeader`: `Title,VrstaDokumenta,Active,Proces`
- Broj zapisa u JSON konfiguraciji: `45`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `VrstaDokumenta` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| Title | VrstaDokumenta |
| --- | --- |
| Ugovor o poslovno-tehničkoj saradnji | Ugovori |
| Ugovor o kupoprodaji | Ugovori |
| Ugovor o donaciji | Ugovori |
| Ugovor o zakupu | Ugovori |
| NDA | Ugovori |
| Ugovor o lizingu | Ugovori |
| Ugovor sa bankama | Ugovori |
| Ugovor o zajmu | Ugovori |
| Aneks ugovora | Ugovori |
| Ugovor o komunalnim uslugama | Ugovori |
| Ugovor o pružanju usluga | Ugovori |
| Ugovor o sponzorstvu | Ugovori |
| Raskid ugovora | Ugovori |
| Ostalo | Ugovori |
| Osnivački akt | Statusna dokumenta |
| Rešenje | Statusna dokumenta |
| Pravilnici i akti | Statusna dokumenta |
| Sertifikati | Statusna dokumenta |
| Ostalo | Statusna dokumenta |
| Tužba | Administrativna dokumenta |
| Dopisi, Obaveštenja, Podnesci | Administrativna dokumenta |
| Odluka | Administrativna dokumenta |
| Rešenje, Presuda, Zaključak | Administrativna dokumenta |
| Punomoćja, Ovlašćena | Administrativna dokumenta |
| Zapisnik | Administrativna dokumenta |
| Obrasci, Prijave, Izveštaji | Administrativna dokumenta |
| Potvrda | Administrativna dokumenta |
| Saglasnost | Administrativna dokumenta |
| Ostalo | Administrativna dokumenta |
| Ugovor o radu | Radni odnosi |
| Dokumentacija uz ugovor o radu | Radni odnosi |
| Prestanak radnog odnosa | Radni odnosi |
| Aneks ugovora o radu | Radni odnosi |
| Godišnji odmori i odsustva | Radni odnosi |
| Potvrde za zaposlene | Radni odnosi |
| Novčane kazne i opomene | Radni odnosi |
| Ostala radno - pravna dokumenta | Radni odnosi |
| Fakture | Finansije |
| Izvodi | Finansije |
| Finansijski izveštaji | Finansije |
| Ostalo | Finansije |
| Avansni račun | Finansije |
| Knjižno zaduženje | Finansije |
| Knjižno odobrenje | Finansije |
| Ostalo | Ostalo |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `45` JSON zapisa.
- Pronađena polja: `Title`, `VrstaDokumenta`.

## Nepoznato / potrebno potvrditi

- Nije potvrđeno u kojoj tačno Power Apps formuli ili Power Automate flow-u se ova konfiguracija koristi.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
