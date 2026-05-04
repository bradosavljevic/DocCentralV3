# DocCentral V3 / e-Pisarnica

## 1. Svrha projekta

DocCentral V3 je white-label Power Platform rešenje za elektronsku pisarnicu, zavođenje, odobravanje, arhiviranje i upravljanje dokumentima kroz Microsoft 365, Power Apps, Power Automate i SharePoint Online.

Rešenje se prilagođava po klijentu kroz konfiguraciju, šifarnike, App Config, Entra grupe i SharePoint strukturu.

## 2. Ključni principi

- SharePoint je primarni data layer.
- Korisnici u SharePoint-u imaju Read Only pristup.
- Sve Create/Edit/Delete operacije idu kroz Power Automate pod servisnim nalogom.
- Canvas aplikacija mora biti responzivna, brza, višejezična i bezbedna.
- Prava pristupa ne smeju biti samo UI filter u Power Apps-u.
- Item/folder/file permissions se dodeljuju na SharePoint nivou.
- Delovodni broj mora biti concurrency-safe.
- Dva korisnika nikada ne smeju dobiti isti delovodni broj.
- Audit/logging se implementira preko SharePoint lista.
- Deployment je ručni import solution-a po tenant-u.
- Migracija nije u scope-u jer novi klijenti kreću iz praznog okruženja.

## 3. Glavni poslovni moduli

1. Novi predmet / unos novog dokumenta
2. Rezervisani delovodni brojevi
3. Dokumenti za odobravanje
4. Podsetnici
5. Arhiviranje
6. Zaključenje godine
7. Partneri
8. Administracija šifarnika i konfiguracija
9. Audit i greške
10. Arhivska knjiga / PDF generisanje

## 4. Statusi dokumenata

Osnovni statusi su:

- Zavedeno
- U odobravanju
- Odobreno
- Odbijeno
- Arhivirano

Napomena: kroz ProcesConfig klijent može definisati dodatne ili prilagođene statuse, ali osnovni statusi predstavljaju bazni model.

## 5. Proces: Unos novog dokumenta

1. Korisnik otvara aplikaciju.
2. Korisnik bira tip dokumenta.
3. Korisnik popunjava obavezna polja.
4. Korisnik dodaje prilog.
5. Korisnik bira da li koristi rezervisani delovodni broj ili generiše sledeći broj u nizu.
6. Ako koristi rezervisani broj, bira datum zavođenja.
7. Ako koristi aktuelni/sledeći broj, datum zavođenja se automatski postavlja na današnji datum.
8. Korisnik klikće "Zavedi".
9. Sistem proverava validaciju.
10. Sistem preuzima ili generiše delovodni broj.
11. Ako broj nije rezervisan, sistem povećava brojač za sledeće zavođenje.
12. Ako je broj rezervisan, sistem nakon uspešnog zavođenja briše broj iz liste rezervisanih brojeva.
13. Sistem kreira SharePoint item.
14. Sistem kreira folder/biblioteku za dokument.
15. Sistem dodeljuje prava.
16. Sistem prikazuje poruku korisniku.

## 6. Šta aplikacija ne radi

- Ne postoji Dashboard.
- Ne postoji ekran "Svi predmeti" u aplikaciji.
- Ne postoji pregled/listanje svih dokumenata u aplikaciji.
- Dokumenti se gledaju direktno u SharePoint listama.
- "Novi predmet" je forma za unos, ne pregled dokumenata.
- SAP import se ignoriše u ovoj fazi dokumentacije i nove verzije.

## 7. Dokumenti u procesu / odobravanje

U aplikaciji korisnik vidi dokumente koje nije odobrio.

U SharePoint listi korisnik vidi dokumente koji su za njega ili dokumente gde je on ili njegova grupa učestvovala u procesu odobravanja.

Odobrenje menja status dokumenta kroz polje `Stanje`.

Proces može biti:

- sekvencijalan po korisnicima
- usmeren na grupu, gde više korisnika dobija informaciju, ali prvi koji odobri završava taj korak

Ako korisnik odbije dokument:

- dokument dobija status `Odbijeno`
- vraća se inicijatoru
- inicijator može izmeniti dokument ili metadata
- inicijator pokreće novi proces odobravanja

## 8. Arhiviranje

Arhiviranje je posebna i bitna funkcionalnost.

Korisnik može da izlista dokumente koji su u statusu `Zavedeno` za tekuću godinu i da ih arhivira uz odgovarajuće arhivske znake.

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen i predstavlja jedinu direktnu putanju arhiviranja.

PDF generisanje u prvoj verziji ulazi samo za Arhivsku knjigu.

## 9. Zaključenje godine

Na kraju godine korisnik može da zaključi godinu.

Uslovi:

- svi dokumenti iz tekuće godine moraju biti u statusu `Arhivirano`
- ne sme postojati dokument u bilo kom drugom statusu
- lista rezervisanih brojeva mora biti prazna
- kreira se nova delovodna knjiga za sledeću godinu
- aktivna godina se menja u App Config
- zaključana godina se nikada ne može ponovo otključati
- nije potrebna dodatna potvrda administratora

## 10. Partneri

Lista Partneri nije deo Administracije, već je zasebna stavka u meniju.

Korisnik koji zavodi dokumente može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

Brisanje partnera ne sme uticati na istorijske dokumente u listi Svi predmeti. Ranije uneti dokumenti moraju zadržati istoriju partnera.

Obrisani partner više ne sme biti dostupan za izbor kod novih dokumenata.

## 11. Rezervisani brojevi

Rezervisane brojeve mogu kreirati korisnici koji zavode dokumente.

Pravila:

- rezervisani broj može da važi samo za jednu godinu
- rezervisani broj može da se iskoristi za bilo koji tip dokumenta
- svi korisnici koji zavode dokumente vide sve rezervisane brojeve
- rezervisani broj može da se izmeni
- nakon rezervacije ne može se ručno obrisati
- briše se automatski tek kada se iskoristi za zavođenje
- lista rezervisanih brojeva mora biti prazna pre zaključenja godine

## 12. Podsetnici

Korisnik može kreirati podsetnik za sebe ili za jednog ili više drugih korisnika.

Podsetnik:

- može biti vezan za dokument/predmet
- šalje email samo jednom
- nema status Aktivan/Poslat/Otkazan
- korisnik može da menja ili briše podsetnik
- email se šalje na dan podsetnika u 08:00
- termin slanja je sistemski konfigurisan, korisnik ne definiše pojedinačno vreme

## 13. Administracija

Administracija prikazuje šifarnike i konfiguracije.

Funkcionalnosti:

- pregled šifarnika
- ručno editovanje pojedinačnih šifarnika
- export pojedinačnog šifarnika u Excel
- export svih šifarnika odjednom u Excel

Partneri nisu deo Administracije.

## 14. Security model

Poznato:

- korisnici imaju Read Only prava nad SharePoint-om
- upisi idu kroz Power Automate
- Power Automate radi pod servisnim nalogom
- pristup dokumentima zavisi od grupa i organizacionih jedinica
- Entra grupa mapping mora biti konfigurabilan po klijentu
- item/folder/file permissions koriste break inheritance
- servisni nalog ima RW
- owner ima RW
- member/viewer imaju Read
- ne postoji posebna admin grupa
- pristup se kontroliše i u Canvas aplikaciji i u SharePoint-u
- backend permissions moraju skrivati file/item, ne samo UI

## 15. Audit / logging

Audit se implementira kroz SharePoint liste.

Obavezno logovati:

- neuspešno kreiranje dokumenta
- pokušaj generisanja delovodnog broja
- uspešno generisanje delovodnog broja
- neuspešno generisanje broja
- korišćenje rezervisanog broja
- arhiviranje
- zaključenje godine
- grešku Power Automate flow-a

## 16. Tehnički standardi

Canvas app mora poštovati sledeće standarde:

- responsive design
- clean App Checker
- formula-level error management
- IfError oko Patch/Flow poziva
- gbl/loc/col naming
- delegabilne formule
- bez live search query-ja na svaki karakter
- bez cross-screen control reference
- bez nested galleries osim ako je opravdano
- standard konektori osim ako korisnik izričito kaže drugačije
- Create/Edit/Delete preko Power Automate
- Read operacije mogu koristiti SharePoint/Office 365/Groups/Microsoft 365 podatke prema potrebi

## 17. Uloga Claude Code-a

Claude Code treba da koristi ovaj repository kao osnovu za:

- razumevanje business procesa
- projektovanje nove Canvas aplikacije
- pisanje tehničke dokumentacije
- predlog strukture ekrana
- implementaciju responsive UX-a
- definisanje Power Automate flow-ova
- definisanje SharePoint lista
- definisanje security modela
- definisanje test matrice
- generisanje development brief-a

Claude Code može sam odlučiti:

- finalni vizuelni dizajn
- finalni broj ekrana
- nazive ekrana
- raspored kontrola
- UX detalje, ako nisu u konfliktu sa business pravilima

Claude Code ne sme menjati:

- concurrency-safe zahtev za delovodni broj
- pravilo da zaključana godina ne može biti otključana
- pravilo da korisnici nemaju direktan Write u SharePoint
- pravilo da Create/Edit/Delete ide preko Power Automate
- pravilo da backend permissions moraju stvarno ograničavati pristup dokumentima
