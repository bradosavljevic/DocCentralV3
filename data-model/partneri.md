# SharePoint lista: Partneri

## Svrha

Lista poslovnih partnera koji se koriste kod zavođenja dokumenata.

Partneri su zasebna stavka u meniju i nisu deo Administracije.

## Funkcionalnosti

Korisnik koji zavodi dokumente može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

## Očekivana polja

Tačna interna imena proveriti iz XML/solution exporta.

Očekivana polja:

- ID
- Title
- PoslovnoIme
- PIB
- MB
- Mesto
- Adresa
- Email
- Telefon
- IsActive
- Created
- Modified
- Author
- Editor

## Pravilo istorije

Brisanje partnera ne sme uticati na istorijske dokumente.

Dokumenti u listi Svi predmeti moraju zadržati partner history.

Preporuka:

- u dokument upisati denormalizovane partner podatke
- koristiti soft delete kada je moguće
- ne dozvoliti izbor obrisanog/deaktiviranog partnera za novi dokument

## Logging

Logovati neuspešne operacije nad partnerima.
