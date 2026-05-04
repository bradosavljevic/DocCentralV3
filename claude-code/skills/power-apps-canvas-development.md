# Skill: Power Apps Canvas Development

## Cilj

Claude Code treba da projektuje novu Canvas aplikaciju na osnovu funkcionalnih zahteva i postojeće dokumentacije, ali bez izmišljanja potvrđenih podataka.

## Obavezna pravila

- Canvas app je frontend/UI sloj.
- Canvas app čita podatke iz SharePoint-a i/ili preko Power Automate flow-ova, u skladu sa dokumentacijom.
- Create/Edit/Delete operacije za poslovne entitete moraju ići preko Power Automate-a.
- Korisnik ne sme dobiti direktan write pristup SharePoint listama samo zato što aplikaciji treba upis.
- Broj ekrana, nazive ekrana i kontrole Claude Code može projektovati sam na osnovu funkcionalnosti nove verzije.
- Postojeće formule, kolekcije, promenljive, navigaciju i UX tok treba čitati iz eksportovanog solution-a kada je dostupan.

## Struktura aplikacije

Preporučene funkcionalne celine:

- Dashboard / početni ekran
- Lista predmeta
- Pregled predmeta
- Novi predmet
- Izmena predmeta
- Zavođenje dokumenta
- Upload / povezivanje dokumenata
- Arhiviranje
- Partneri
- Konfiguracije / šifarnici, samo za ovlašćene korisnike ako se kasnije definiše
- Greške / status operacija, ako postoji centralni log

## App.OnStart / App.Formulas

Claude Code mora definisati ili dokumentovati:

- trenutnog korisnika;
- korisničke grupe;
- organizacione jedinice korisnika;
- lokalizaciju;
- format datuma i brojeva;
- inicijalno učitavanje konfiguracije iz `App Config`;
- inicijalne kolekcije;
- globalne boje / theme vrednosti;
- feature flags ako postoje u `App Config`.

## Kolekcije

Kolekcije treba koristiti za:

- konfiguracije i šifarnike;
- trenutnog korisnika i njegove grupe;
- lookup podatke koji se često koriste;
- read-only podatke za prikaz;
- privremeni UI state.

Kolekcije ne smeju biti jedini izvor istine za kritične operacije kao što su delovodni broj, prava pristupa i finalni status dokumenta.

## Zabranjeno

- Generisanje delovodnog broja samo lokalno u Canvas aplikaciji.
- Oslanjanje na `Max(ID)+1` ili lokalnu kolekciju za jedinstveni poslovni broj.
- Direktan Patch u SharePoint za krajnje korisnike kada je pravilo da se piše kroz Power Automate.
- Sakrivanje podataka samo kroz UI bez backend permission modela.
