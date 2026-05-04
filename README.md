# DocCentral V3 / e-Pisarnica

## Kreirani Power Platform objekti / obavezna imena

Ova dokumentacija je usklađena sa objektima koji su već ručno kreirani u Power Platform solution-u.

### Solution

| Stavka | Vrednost |
|---|---|
| Solution display name | `DocCentralV3` |
| Solution name | `DocCentralV3` |
| Publisher | `GoProDocCentral` |
| Version | `3.0.0.0` |
| Package type | `Unmanaged` u development okruženju |

### Canvas app

| Stavka | Vrednost |
|---|---|
| Display name | `DocCentralV3` |
| Logical/name | `gpdoccen_doccentralv3_d98ba` |

### Connection references

| Display name | Logical/name | Namena |
|---|---|---|
| `CR_DocCentralV3_SharePoint` | `gpdoccen_CR_DocCentralV3_SharePoint` | SharePoint liste/biblioteke |
| `CR_DocCentralV3_Outlook` | `gpdoccen_CR_DocCentralV3_Outlook` | Email obaveštenja i podsetnici |
| `CR_DocCentralV3_Office365Users` | `gpdoccen_CR_DocCentralV3_Office365Users` | Korisnici/profili |
| `CR_DocCentralV3_Office365Groups` | `gpdoccen_CR_DocCentralV3_Office365Groups` | Entra/M365 grupe |
| `CR_DocCentralV3_OneDrive` | `gpdoccen_CR_DocCentralV3_OneDrive` | Privremeni fajlovi/Excel/PDF scenariji ako je potrebno |
| `CR_DocCentralV3_Excel` | `gpdoccen_CR_DocCentralV3_Excel` | Export u Excel |

### Environment variables

| Display name | Logical/name | Tip vrednosti | Namena |
|---|---|---|---|
| `EV_DocCentralV3_SharePointSite` | `gpdoccen_EV_DocCentralV3_SharePointSite` | SharePoint site | Root SharePoint site za rešenje |
| `EV_DocCentralV3_lstSviPredmeti` | `gpdoccen_EV_DocCentralV3_lstSviPredmeti` | SharePoint list | Glavna lista dokumenata/predmeta |
| `EV_DocCentralV3_lstPartneri` | `gpdoccen_EV_DocCentralV3_lstPartneri` | SharePoint list | Partneri |
| `EV_DocCentralV3_lstAppConfig` | `gpdoccen_EV_DocCentralV3_lstAppConfig` | SharePoint list | Šifarnici i konfiguracija |
| `EV_DocCentralV3_lstRezervisaniBrojevi` | `gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi` | SharePoint list | Rezervisani delovodni brojevi |
| `EV_DocCentralV3_lstPodsetnici` | `gpdoccen_EV_DocCentralV3_lstPodsetnici` | SharePoint list | Podsetnici |
| `EV_DocCentralV3_lstAuditLog` | `gpdoccen_EV_DocCentralV3_lstAuditLog` | SharePoint list | Audit/log lista |
| `EV_DocCentralV3_docDokumenti` | `gpdoccen_EV_DocCentralV3_docDokumenti` | SharePoint document library | Dokumenti/prilozi |
| `EV_DocCentralV3_docEmailDocs` | `gpdoccen_EV_DocCentralV3_docEmailDocs` | SharePoint document library | Email dokumenti |
| `EV_DocCentralV3_docExports` | `gpdoccen_EV_DocCentralV3_docExports` | SharePoint document library | Excel/PDF export fajlovi |

### Cloud flow nazivi

Svi flow-ovi moraju koristiti prefix `CF_`, ne `PA_`.

| Flow display name |
|---|
| `CF_DocCentralV3_CreateDocument` |
| `CF_DocCentralV3_GenerateRegistryNumber` |
| `CF_DocCentralV3_UseReservedNumber` |
| `CF_DocCentralV3_CreateDocumentFolder` |
| `CF_DocCentralV3_AssignPermissions` |
| `CF_DocCentralV3_SendForApproval` |
| `CF_DocCentralV3_ProcessApprovalResponse` |
| `CF_DocCentralV3_SendReminders` |
| `CF_DocCentralV3_ArchiveDocument` |
| `CF_DocCentralV3_CloseRegistryYear` |
| `CF_DocCentralV3_ExportAppConfig` |
| `CF_DocCentralV3_GenerateArchiveBookPdf` |
| `CF_DocCentralV3_LogEvent` |

### Pravilo

Claude Code mora koristiti ova postojeća imena i ne sme predlagati nova imena za već kreirane solution objekte, osim ako korisnik izričito traži promenu.

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

## 18. PACode folder pravilo

Svi fajlovi sa kodom koje Claude Code generiše moraju biti smešteni u folder `PACode`.

Ovo uključuje:

- Power Apps formule
- Power Fx isečke
- Power Automate JSON definicije ili pseudocode
- SharePoint REST/Graph primere
- PowerShell skripte
- CLI komande
- test skripte
- helper funkcije
- bilo koji drugi fajl koji predstavlja izvršivi, polu-izvršivi ili tehnički kod

Dokumentacija ostaje u postojećim folderima kao što su `business`, `architecture`, `data-model`, `power-apps`, `power-automate`, `security`, `testing`, `templates`, `skills` i drugi.

Kod ne sme biti razbacan kroz root ili dokumentacione foldere, osim ako je u pitanju kratki primer unutar `.md` dokumenta. Svi zasebni code fajlovi moraju ići u `PACode`.
