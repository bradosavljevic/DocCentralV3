# 10 — Known Issues & Technical Debt

## 1. Svrha dokumenta

Ovaj dokument opisuje poznate rizike, tehnički dug i otvorena pitanja u postojećem rešenju **DocCentral v6.0**.

Dokument služi kao osnova za:

- stabilizaciju postojećeg rešenja
- planiranje enterprise unapređenja
- pripremu nove verzije aplikacije
- definisanje backlog stavki za razvoj
- smanjenje rizika kod zavođenja dokumenata

---

## 2. Status analize

### Potvrđeno

Na osnovu dostavljenih informacija i SharePoint REST metadata odgovora potvrđeno je sledeće:

- Rešenje koristi **Power Apps Canvas App**.
- Početni ekran aplikacije je `scrHome`.
- Postoji posebna funkcionalnost za zavođenje dokumenta.
- Pregled dokumenata se radi direktno kroz SharePoint listu, a ne kroz poseban ekran u aplikaciji.
- Email dokumenti se obrađuju posebnom funkcionalnošću gde Power Automate čita shared mailbox i zavodi dokumente iz emailova sa attachmentom.
- Export služi za izvoz šifarnika i konfiguracije iz `AppConfig` liste.
- Svi šifarnici i konfiguracioni JSON podaci nalaze se u `AppConfig` listi.
- Upload dokumenta ide preko Power Automate flow-a `Upload Doc`.
- Dodela / dodavanje delovodnog broja za novi dokument ide preko posebnog flow-a zbog race condition problema.
- Flow koji se koristi za ovu svrhu je `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`.

### Delimično potvrđeno

- Lista `RezervisaniBrojevi` postoji i sadrži kolonu `RezervisaniBroj`.
- Na osnovu naziva liste i polja može se zaključiti da učestvuje u logici rezervacije brojeva, ali kompletna flow logika nije analizirana iz exportovanog flow JSON-a.

### Nepoznato

- Kompletna Power Apps formula logika nije dostupna.
- Kompletna Power Automate definicija flow-ova nije dostupna.
- Nije potvrđeno da li SharePoint kolone imaju uključene unique constraints.
- Nije potvrđeno da li postoji centralni audit log za zavođenje dokumenata.
- Nije potvrđeno da li postoji retry logika u svim flow-ovima.
- Nije potvrđeno da li postoje environment variables i connection references u managed solution-u.
- Nije potvrđeno da li postoji posebna lista za greške i tehničke logove.

---

## 3. Najvažniji poznati rizik

## 3.1 Dupliranje delovodnog broja

### Opis

Najkritičniji poslovni i tehnički rizik je mogućnost da dva korisnika istovremeno zavode dokument i dobiju isti delovodni broj.

Ovo je enterprise-critical zahtev.

Sistem mora dozvoliti da više korisnika istovremeno zavodi dokumente, ali nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

### Uticaj

Ako dođe do dupliranja delovodnog broja:

- narušava se integritet pisarnice
- dokumenti gube jednoznačnu identifikaciju
- otežava se audit
- mogu nastati pravni i operativni problemi
- sistem postaje nepouzdan za produkcionu upotrebu

### Trenutno stanje

Potvrđeno je da postoji poseban flow za rešavanje race condition problema:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Takođe je potvrđeno da se upload dokumenta radi preko flow-a:

```text
Upload Doc
```

### Tehnički dug

Nije dovoljno da Canvas aplikacija sama izračuna sledeći broj.

Ako se sledeći broj računa u Canvas aplikaciji na osnovu lokalne kolekcije ili poslednjeg pročitanog zapisa iz SharePoint liste, sistem nije concurrency-safe.

### Preporuka

Generisanje i finalna dodela delovodnog broja mora biti centralizovana u backend logici.

Minimalni enterprise zahtevi:

- server-side generisanje broja
- atomic update ili optimistic locking
- retry logika
- unique constraint na finalnom delovodnom broju
- audit log za svaki pokušaj rezervacije
- kontrolisana greška prema Canvas aplikaciji
- zabrana da Canvas aplikacija sama garantuje jedinstvenost broja

---

## 4. Rizik: poslovna logika u Canvas aplikaciji

### Opis

Canvas aplikacija treba da bude presentation i orchestration layer, a ne centralno mesto za kritičnu poslovnu logiku.

### Problem

Ako Canvas aplikacija sadrži previše poslovne logike:

- teže se održava
- teže se testira
- teže se verzioniše
- povećava se rizik od grešaka
- povećava se rizik od razlike između korisničkih sesija
- povećava se rizik od race condition problema

### Dozvoljena uloga Canvas aplikacije

Canvas aplikacija sme da:

- prikazuje forme
- validira osnovna obavezna polja
- pokreće Power Automate flow-ove
- prikazuje rezultat korisniku
- prikazuje status obrade
- učitava konfiguraciju iz `AppConfig`

Canvas aplikacija ne treba da:

- samostalno generiše konačni delovodni broj
- garantuje jedinstvenost delovodnog broja
- direktno upisuje kritične podatke bez backend validacije
- sadrži hardkodovane šifarnike ako već postoji `AppConfig`
- sadrži poslovna pravila koja moraju biti centralizovana

---

## 5. Rizik: SharePoint kao primarni data layer

### Opis

SharePoint Online se koristi kao primarni data layer i storage layer.

To je prihvatljivo za mnoga Power Platform rešenja, ali za enterprise pisarnicu postoje ograničenja.

### Rizici

Potencijalni rizici:

- list view threshold
- delegacija u Power Apps aplikaciji
- performanse kod velikog broja dokumenata
- throttling SharePoint konektora
- item-level permissions kompleksnost
- spori upiti nad neindeksiranim kolonama
- nedostatak transakcione baze
- race condition kod paralelnih upisa
- kompleksno logovanje i audit
- zavisnost od Power Automate trajanja i timeout-a

### Preporuka

Za postojeću verziju:

- indeksirati ključne kolone
- koristiti delegabilne filtere
- smanjiti direktne Canvas upite
- koristiti Power Automate za kontrolisane upise
- uvesti centralni audit log
- uvesti centralni error log

Za novu enterprise verziju razmotriti:

- Dataverse za transakcione entitete
- Azure SQL za atomic counter logiku
- Azure Function / Logic Apps za concurrency-safe numbering service
- Application Insights za monitoring
- centralni API layer

---

## 6. Rizik: AppConfig kao centralna konfiguracija

### Opis

`AppConfig` lista se koristi za šifarnike i konfiguracione JSON podatke.

Ovo je fleksibilan obrazac, ali može postati tehnički dug ako nije strogo kontrolisan.

### Potvrđena polja

Identifikovana su sledeća važna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Title` | Single line of text | Naziv konfiguracije |
| `Config` | Multiple lines of text | JSON konfiguracija |
| `ColumnHeader` | Multiple lines of text | Konfiguracija kolona / prikaza |

### Rizici

- nevalidan JSON može srušiti deo aplikacije
- nema potvrde da postoji schema validation
- promena konfiguracije može uticati na produkciju bez deployment procesa
- nema potvrde da postoji istorija promene konfiguracije
- nema potvrde da postoji approval za promene konfiguracije
- moguće je da Canvas aplikacija očekuje određenu strukturu JSON-a koja nije dokumentovana

### Preporuka

Za `AppConfig` treba uvesti:

- JSON schema dokumentaciju
- primer za svaki tip konfiguracije
- validaciju JSON-a pre korišćenja
- verzionisanje konfiguracije
- polje `IsActive`
- polje `ConfigType`
- polje `Version`
- approval proces za izmene
- export/import proces za konfiguracije
- fallback vrednosti u Canvas aplikaciji

---

## 7. Rizik: Email intake proces

### Opis

Email dokumenti se obrađuju kroz posebnu funkcionalnost gde Power Automate čita shared mailbox i zavodi dokumente iz emailova sa attachmentom.

Biblioteka:

```text
EmailDocuments
```

Identifikovana polja:

| Internal name | Tip |
|---|---|
| `FileLeafRef` | File |
| `Title` | Single line of text |
| `_ExtendedDescription` | Multiple lines of text |
| `PosiljalacEmail` | Single line of text |
| `Posiljalac` | Single line of text |
| `MediaServiceImageTags` | Managed Metadata |
| `ContentType` | Computed |

### Rizici

- nije potvrđeno kako se obrađuju emailovi bez attachmenta
- nije potvrđeno kako se obrađuje više attachmenta u jednom emailu
- nije potvrđeno da li postoji deduplikacija emailova
- nije potvrđeno da li se čuva Message ID
- nije potvrđeno da li se čuva originalni subject
- nije potvrđeno da li se čuva originalni sender
- nije potvrđeno da li postoji retry ako upload attachmenta ne uspe
- nije potvrđeno da li postoji error log za neuspešne email obrade

### Preporuka

Email intake proces treba da ima:

- čuvanje originalnog `MessageId`
- čuvanje `Subject`
- čuvanje `ReceivedDateTime`
- čuvanje `SenderEmail`
- čuvanje broja attachmenta
- status obrade
- error message
- retry count
- deduplication key
- audit log
- karantin za neuspešno obrađene emailove

---

## 8. Rizik: Export proces

### Opis

Export funkcionalnost služi za export šifarnika i konfiguracije iz `AppConfig` liste.

Biblioteka:

```text
Exports
```

Identifikovana polja:

| Internal name | Tip |
|---|---|
| `FileLeafRef` | File |
| `Title` | Single line of text |
| `_ExtendedDescription` | Multiple lines of text |
| `ContentType` | Computed |

### Rizici

- nije potvrđeno koji format exporta se koristi
- nije potvrđeno da li export uključuje sve konfiguracije
- nije potvrđeno da li export ima verziju
- nije potvrđeno da li se export može vratiti nazad kao import
- nije potvrđeno da li postoji kontrola ko može da pokrene export
- nije potvrđeno da li export sadrži osetljive podatke

### Preporuka

Export treba standardizovati:

- format: JSON ili ZIP
- naziv fajla sa datumom i verzijom
- export manifest
- lista uključenih konfiguracija
- checksum
- korisnik koji je pokrenuo export
- datum exporta
- mogućnost import validacije
- permission kontrola

---

## 9. Rizik: Nedovoljno dokumentovan Power Automate sloj

### Opis

Power Automate flow-ovi su ključni backend sloj rešenja.

Potvrđeni flow-ovi:

```text
Upload Doc
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

### Rizici

- nisu dostupne kompletne definicije flow-ova
- nije poznata trigger logika svih flow-ova
- nije poznato koji flow-ovi se pozivaju iz Canvas aplikacije
- nije poznato koji flow-ovi se pokreću automatski
- nije poznato koje konekcije koriste flow-ovi
- nije poznato da li postoje child flow-ovi
- nije poznato da li se koristi service account
- nije poznato da li postoji error handling standard

### Preporuka

Za svaki flow treba dokumentovati:

- naziv flow-a
- trigger
- ulazne parametre
- izlazne parametre
- konektore
- connection reference
- environment variables
- SharePoint liste/biblioteke koje koristi
- retry policy
- error handling
- logging
- timeout rizike
- vlasnika flow-a
- poslovnu namenu

---

## 10. Rizik: Security i permissions model

### Opis

Rešenje koristi SharePoint liste i biblioteke kao data layer.

Kod ovakvog modela permissions arhitektura je kritična.

### Rizici

- nije potvrđeno da li korisnici imaju direktna prava na SharePoint liste
- nije potvrđeno da li svi upisi idu preko Power Automate flow-ova
- nije potvrđeno da li flow-ovi rade pod service account-om
- nije potvrđeno da li postoje item-level permissions
- nije potvrđeno kako se kontroliše pristup bibliotekama
- nije potvrđeno ko može da menja `AppConfig`
- nije potvrđeno ko može da vidi `RezervisaniBrojevi`
- nije potvrđeno ko može da pokrene export

### Preporuka

Enterprise model treba da bude:

- korisnici imaju read-only pristup SharePoint listama gde je moguće
- svi kritični upisi idu preko Power Automate flow-ova
- flow-ovi rade pod kontrolisanim service account-om
- `AppConfig` ima strogo ograničen edit pristup
- `RezervisaniBrojevi` nije direktno vidljiv običnim korisnicima
- export mogu pokretati samo ovlašćeni administratori
- sve promene se loguju

---

## 11. Rizik: Nedostatak centralnog logging modela

### Opis

Kod poslovno kritične aplikacije kao što je elektronska pisarnica, logging nije opcionalan.

### Rizici

Bez centralnog logovanja teško je odgovoriti na pitanja:

- ko je pokušao da zavede dokument
- kada je pokušao
- koji delovodni broj je rezervisan
- da li je rezervacija uspela
- da li je upload dokumenta uspeo
- koji flow je pukao
- koji payload je poslat
- koji error je vraćen
- da li je bilo retry pokušaja
- da li je došlo do konflikta

### Preporuka

Uvesti centralnu listu ili tabelu:

```text
DocCentralLogs
```

Minimalna polja:

| Polje | Tip |
|---|---|
| `Title` | Text |
| `CorrelationId` | Text |
| `ProcessName` | Text |
| `FlowName` | Text |
| `Source` | Choice |
| `StartedAt` | DateTime |
| `FinishedAt` | DateTime |
| `Status` | Choice |
| `UserEmail` | Text |
| `DocumentId` | Text |
| `DelovodniBroj` | Text |
| `PayloadJson` | Multiple lines of text |
| `ResponseJson` | Multiple lines of text |
| `ErrorMessage` | Multiple lines of text |
| `RetryCount` | Number |

---

## 12. Rizik: Nedostatak correlation ID obrasca

### Opis

Kod procesa koji uključuje Canvas App, Power Automate i SharePoint, potrebno je imati isti correlation ID kroz ceo proces.

### Rizici

Bez correlation ID-a teško je povezati:

- klik korisnika u aplikaciji
- poziv flow-a
- kreiranje dokumenta
- rezervaciju broja
- upload fajla
- grešku u SharePoint-u

### Preporuka

Canvas aplikacija treba da generiše ili primi `CorrelationId`.

Flow-ovi treba da ga prosleđuju kroz sve akcije.

Svi logovi treba da imaju isti `CorrelationId`.

Primer:

```text
DC-2026-04-30-USER-000001
```

---

## 13. Rizik: Naming convention nije kompletno dokumentovan

### Opis

Deo naming convention-a je vidljiv kroz postojeće nazive:

```text
scrHome
Upload Doc
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
AppConfig
RezervisaniBrojevi
EmailDocuments
Exports
```

### Rizici

Ako naming convention nije dosledan:

- teže je održavanje
- teže je onboarding novih developera
- teže je automatsko generisanje dokumentacije
- teže je povezati aplikaciju, flow-ove i SharePoint entitete

### Preporuka

Definisati standard:

#### Power Apps ekrani

```text
scrHome
scrDocumentEntry
scrAdmin
scrConfig
```

#### Power Apps kontrole

```text
btnSubmit
galDocuments
frmDocument
cmbDocumentType
txtComment
```

#### Power Automate flow-ovi

```text
CF_DocCentral21_<Action><Context>
```

Primer:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

#### SharePoint liste

```text
DC_<Domain>_<Entity>
```

#### SharePoint biblioteke

```text
DC_Documents
DC_EmailDocuments
DC_Exports
```

---

## 14. Rizik: Nedostatak test scenarija

### Opis

Za poslovno kritično rešenje neophodni su formalni test scenariji.

### Kritični testovi

Obavezno testirati:

- jedan korisnik zavodi jedan dokument
- dva korisnika istovremeno zavode dokumente
- pet korisnika istovremeno zavodi dokumente
- upload dokumenta uspeva
- upload dokumenta ne uspeva
- flow za broj vraća grešku
- SharePoint vraća conflict
- korisnik nema prava
- email sa jednim attachmentom
- email sa više attachmenta
- email bez attachmenta
- export konfiguracije
- nevalidan JSON u `AppConfig`
- retry nakon konflikta
- duplikat delovodnog broja

### Posebno važan test

Najvažniji test:

```text
Concurrency test — više paralelnih korisnika pokreće zavođenje dokumenta u isto vreme.
```

Očekivani rezultat:

```text
Svaki dokument dobija jedinstven delovodni broj.
Nijedan broj se ne duplira.
Svi pokušaji su logovani.
Greške su kontrolisane.
```

---

## 15. Prioriteti tehničkog duga

| Prioritet | Stavka | Razlog |
|---|---|---|
| P0 | Concurrency-safe delovodni broj | Kritičan poslovni zahtev |
| P0 | Unique constraint na finalni delovodni broj | Sprečava duplikate |
| P0 | Audit log za zavođenje | Potreban za dokazivost |
| P1 | Centralni error log | Potreban za podršku |
| P1 | AppConfig schema validation | Sprečava greške konfiguracije |
| P1 | Dokumentacija flow-ova | Potrebna za održavanje |
| P1 | Security model dokumentacija | Potrebna za enterprise upotrebu |
| P2 | Naming convention | Poboljšava održavanje |
| P2 | Export/import konfiguracije | Poboljšava administraciju |
| P2 | Test scenariji | Poboljšava kvalitet |

---

## 16. Preporučeni backlog

### P0 — Kritično

- Implementirati ili potvrditi atomic numbering servis.
- Postaviti unique constraint na finalni delovodni broj.
- Uvesti audit log za svaki pokušaj zavođenja.
- Testirati paralelno zavođenje dokumenata.
- Dokumentovati flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`.

### P1 — Visok prioritet

- Dokumentovati `Upload Doc` flow.
- Uvesti centralni error log.
- Definisati JSON schema za `AppConfig`.
- Ograničiti edit prava nad `AppConfig`.
- Ograničiti direktan pristup listi `RezervisaniBrojevi`.
- Dokumentovati permissions model.

### P2 — Srednji prioritet

- Standardizovati naming convention.
- Dodati export manifest.
- Dodati import validation.
- Dodati test plan.
- Dodati troubleshooting dokumentaciju.

---

## 17. Otvorena pitanja

Za potpunu analizu potrebno je dodatno potvrditi:

1. Da li kolona finalnog delovodnog broja ima uključeno `EnforceUniqueValues`?
2. Da li flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` koristi SharePoint ETag ili neki drugi locking mehanizam?
3. Da li `RezervisaniBrojevi` ima unique constraint na `RezervisaniBroj`?
4. Da li postoji posebna lista za logove?
5. Da li postoji posebna lista za greške?
6. Da li `Upload Doc` vraća Canvas aplikaciji status i grešku?
7. Da li se svaki flow poziv vezuje za `CorrelationId`?
8. Da li korisnici imaju read-only prava na SharePoint liste?
9. Da li svi kritični upisi idu preko Power Automate flow-ova?
10. Da li flow-ovi rade pod service account-om?
11. Da li postoji ALM proces za promene `AppConfig` konfiguracije?
12. Da li postoji proces za backup/export konfiguracije?
13. Da li email intake čuva originalni email metadata?
14. Da li se obrađuju emailovi sa više attachmenta?
15. Da li postoji deduplikacija email poruka?

---

## 18. Zaključak

Najvažniji tehnički dug u rešenju **DocCentral v6.0** je kontrola jedinstvenog delovodnog broja u scenariju paralelnog rada više korisnika.

Postojeća arhitektura već prepoznaje ovaj problem, jer postoji poseban flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Međutim, za enterprise nivo potrebno je formalno potvrditi i dokumentovati:

- kako flow sprečava race condition
- da li postoji locking
- da li postoji retry
- da li postoji unique constraint
- da li postoji audit log
- kako se greška vraća korisniku

Dok se ovo ne potvrdi, concurrency-safe numbering ostaje najvažniji P0 rizik i najvažnija stavka tehničkog duga.
