# Proces: Podsetnici

## Cilj

Omogućiti korisniku da kreira podsetnik za sebe ili za druge korisnike vezano za dokument/predmet ili internu radnju.

## Pravila

- Podsetnik može kreirati korisnik za sebe.
- Podsetnik može kreirati korisnik za jednog ili više drugih korisnika.
- Korisnik može menjati podsetnik.
- Korisnik može brisati podsetnik.
- Podsetnik šalje email samo jednom.
- Ne postoji status podsetnika kao Aktivan, Poslat ili Otkazan.
- Email se šalje na dan definisan u podsetniku.
- Email se šalje u 08:00 ujutro.
- Vreme slanja je sistemski konfigurisano.
- Korisnik ne definiše individualno vreme slanja.

## Implementacija

Preporuka je da postoji SharePoint lista za podsetnike.

Power Automate scheduled flow treba jednom dnevno da:

1. pronađe podsetnike za današnji datum
2. pošalje email korisnicima
3. evidentira uspešno ili neuspešno slanje u log listu
4. obezbedi da isti podsetnik ne pošalje email više puta

Napomena: iako nema poslovni status podsetnika, tehnička implementacija može imati interno polje za sprečavanje duplog slanja ako je to potrebno, ali to polje ne treba izlagati korisniku kao poslovni status.
