# DocCentral V3 / e-Pisarnica

## Status dokumentacije

Ovaj repozitorijum sadrži poslovnu, tehničku i razvojnu dokumentaciju za novu verziju DocCentral / e-pisarnica rešenja.

Cilj dokumentacije je da bude osnova za:

- razvoj nove Power Apps Canvas aplikacije
- razvoj Power Automate flow-ova
- definisanje SharePoint data modela
- pripremu Claude Code development brief-a
- standardizaciju white-label implementacije po klijentu
- buduću GitHub dokumentaciju projekta

## Osnovni koncept rešenja

DocCentral je white-label e-pisarnica / elektronska evidencija dokumenata koja se prilagođava po klijentu.

Rešenje koristi:

- Power Apps Canvas aplikaciju kao korisnički interfejs
- Power Automate kao servisni sloj za upise, prava i poslovnu logiku
- SharePoint Online liste i biblioteke kao data/storage sloj
- Microsoft 365 / Entra ID grupe za kontrolu pristupa
- App Config listu kao centralni izvor šifarnika i konfiguracija

## Ključni arhitektonski princip

Korisnici imaju Read Only prava nad SharePoint sadržajem.

Svi upisi, izmene, dodela prava, generisanje delovodnog broja i sistemske operacije moraju ići kroz Power Automate flow-ove koji rade pod servisnim nalogom.

Aplikacija ne sme da se oslanja samo na UI filtriranje. Backend prava na SharePoint item/folder/file nivou moraju stvarno ograničavati pristup.

## Važni poslovni procesi

### Unos novog dokumenta

Korisnik otvara aplikaciju, bira tip dokumenta, popunjava obavezna polja, dodaje prilog, bira da li koristi rezervisani delovodni broj ili sledeći broj u nizu, zatim klikće `Zavedi`.

Ako se koristi rezervisani broj, korisnik bira datum zavođenja.

Ako se koristi sledeći broj u nizu, datum zavođenja se automatski postavlja na današnji datum.

Sistem validira unos, preuzima ili generiše delovodni broj, kreira SharePoint item, kreira folder/biblioteku za dokument, dodeljuje prava i prikazuje poruku korisniku.

### Arhiviranje

Arhiviranje je posebna funkcionalnost / ekran.

Korisnik može da izlista sve dokumente u statusu `Zavedeno` za tekuću godinu i da ih arhivira sa odgovarajućim arhivskim znacima.

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen i to je jedini direktni put arhiviranja.

### Zaključenje godine

Na kraju godine moguće je zaključiti godinu samo ako su ispunjeni svi uslovi:

- svi dokumenti iz tekuće godine su u statusu `Arhivirano`
- ne postoji nijedan dokument u drugom statusu
- lista rezervisanih brojeva je prazna
- kreira se nova delovodna knjiga za sledeću godinu
- aktivna godina se menja u App Config

Zaključana godina se nikada ne može ponovo otključati.

## Statusi dokumenata

Osnovni statusi su:

- Zavedeno
- U odobravanju
- Odobreno
- Odbijeno
- Arhivirano

Kroz `ProcesConfig` klijent može definisati i dodatne / prilagođene statuse.

## Approval proces

Dokument ide na odobrenje jednom korisniku u jednom koraku.

Ako ima više korisnika, odobravanje je sekvencijalno.

Postoji mogućnost slanja koraka na grupu. Tada više korisnika dobija obaveštenje, ali prvi korisnik koji odobri završava taj status / korak.

Odobrenje menja status dokumenta kroz polje `Stanje`.

Ako je dokument odbijen, dobija status `Odbijeno`, vraća se inicijatoru i inicijator može ponovo pokrenuti approval nakon izmene dokumenta ili metadata polja.

## Ekrani aplikacije

U aplikaciji ne postoji Dashboard.

U aplikaciji ne postoji ekran `Svi predmeti` za pregled svih dokumenata.

Dokumenti se pregledaju direktno u SharePoint listama.

`Novi predmet` je forma za unos novog dokumenta.

Aplikacija treba da sadrži funkcionalnosti za:

- unos novog dokumenta
- arhiviranje
- zaključenje godine
- podsetnike
- dokumente koje korisnik treba da odobri
- administraciju šifarnika
- partnere kao posebnu stavku menija

Finalni broj ekrana, nazive ekrana i kontrole treba da odredi Claude Code na osnovu funkcionalnih zahteva i standarda iz ovog repozitorijuma.

## Partneri

Partneri nisu deo Administracije, već posebna stavka u meniju.

Korisnik koji zavodi dokumente može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

Brisanje partnera ne sme uticati na istorijske dokumente u `Svi predmeti`.

Obrisani partner ne sme biti dostupan za izbor u novim dokumentima.

## Rezervisani delovodni brojevi

Korisnik koji zavodi dokumente može da kreira i koristi rezervisane brojeve.

Rezervisani broj:

- važi samo za jednu godinu
- može se koristiti za bilo koji tip dokumenta
- vide ga svi korisnici koji imaju pravo zavođenja
- može se izmeniti nakon kreiranja
- ne može se ručno obrisati nakon rezervacije
- briše se automatski kada se iskoristi

## Podsetnici

Korisnik može kreirati podsetnik za sebe ili za jednog / više drugih korisnika.

Podsetnik se šalje emailom jednom, na datum podsetnika, u 08:00 ujutro ili prema sistemskoj konfiguraciji.

Korisnik ne definiše individualno vreme podsetnika.

Korisnik može menjati i brisati podsetnike.

Ne postoji status podsetnika kao `Aktivan`, `Poslat` ili `Otkazan`, osim ako se naknadno uvede kao tehničko poboljšanje.

## Logging i audit

Potrebna je sistemska log lista.

Obavezno logovati:

- neuspešno kreiranje dokumenta
- neuspešan pokušaj generisanja delovodnog broja
- uspešno generisan delovodni broj
- neuspešno generisanje broja
- korišćenje rezervisanog broja
- arhiviranje
- zaključenje godine
- grešku Power Automate flow-a

## Code quality standard

Nova aplikacija mora poštovati sledeće standarde:

- responsive design
- čist App Checker
- formula-level error management
- `IfError` oko Patch / Flow poziva
- `gbl`, `loc`, `col` naming konvencije
- delegabilne formule
- bez live search query-ja na svaki karakter
- bez cross-screen referenciranja kontrola
- bez nested galleries osim ako je opravdano

## Struktura repozitorijuma

```text
.
├── README.md
├── architecture/
├── business/
├── checklists/
├── claude-code/
├── data-model/
├── processes/
├── security/
├── skills/
└── templates/
```

## Sporno / nepoznato

Sledeće stavke ostaju otvorene ili se čitaju iz solution exporta:

- formule iz Canvas aplikacije
- kompletan inventar flow-ova
- connection references
- environment variables
- detaljna App Config upotreba po ekranima
- konačan deployment model po tenantima
- konačna mapa Entra grupa po klijentu
- detaljan PDF generation mehanizam
- migration strategija za postojeće produkcione podatke
