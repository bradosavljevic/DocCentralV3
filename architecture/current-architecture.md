# Current Architecture — DocCentral v6.0

## 1. Svrha dokumenta

Ovaj dokument opisuje trenutno poznatu arhitekturu rešenja **DocCentral v6.0** na osnovu dostavljenih informacija, SharePoint REST metadata odgovora i dodatnih pojašnjenja o aplikaciji.

Cilj dokumenta je da prikaže:

- trenutne komponente sistema
- odnos između Power Apps, Power Automate i SharePoint Online
- tokove podataka
- poznate arhitektonske odluke
- rizike trenutne arhitekture
- oblasti koje su još uvek nepoznate

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Power Apps tip aplikacije | POTVRĐENO |
| Početni ekran | POTVRĐENO |
| SharePoint liste i biblioteke | DELIMIČNO POTVRĐENO |
| AppConfig koncept | POTVRĐENO |
| Zavođenje dokumenata | DELIMIČNO POTVRĐENO |
| Power Automate flow za upload dokumenta | POTVRĐENO |
| Power Automate flow za dodelu delovodnog broja | POTVRĐENO |
| Email intake proces | POTVRĐENO konceptualno |
| Export proces | POTVRĐENO konceptualno |
| Security model | DELIMIČNO POZNATO |
| Kompletna Power Apps logika | NEPOZNATO |
| Kompletna Power Automate logika | NEPOZNATO |

---

## 3. Visok nivo trenutne arhitekture

Trenutna arhitektura rešenja može se opisati kao Power Platform rešenje sa sledećim glavnim slojevima:

```text
Korisnik
   |
   v
Power Apps Canvas App
   |
   | poziva flow-ove / prikazuje forme / validira unos
   v
Power Automate
   |
   | kreira dokumente / dodeljuje delovodni broj / obrađuje email / exportuje podatke
   v
SharePoint Online
   |
   | liste + biblioteke + konfiguracija + dokumenti
   v
DocCentral podaci i dokumenti
```

---

## 4. Glavne komponente sistema

### 4.1 Power Apps Canvas App

Aplikacija je potvrđena kao **Power Apps Canvas App**.

Poznato:

- aplikacija ima početni ekran `scrHome`
- `scrHome` je početni ekran
- postoji poseban ekran za zavođenje dokumenta
- pregled dokumenata nije poseban ekran u aplikaciji
- pregled dokumenata se radi direktno u SharePoint listi
- Canvas aplikacija koristi konfiguraciju iz SharePoint liste `AppConfig`
- upload dokumenta ide preko Power Automate flow-a `Upload Doc`
- dodela delovodnog broja ide preko posebnog flow-a zbog race condition problema

Nepoznato:

- kompletan spisak ekrana
- nazivi svih kontrola
- formule po ekranima
- kolekcije koje se kreiraju pri pokretanju aplikacije
- kompletan `OnStart` / `App.Formulas` model
- kompletna navigaciona logika
- sva validaciona pravila

---

### 4.2 Power Automate

Power Automate predstavlja backend sloj za kritične operacije.

Potvrđeni flow-ovi / funkcionalne oblasti:

| Flow / oblast | Status | Namena |
|---|---|---|
| `Upload Doc` | POTVRĐENO | Upload dokumenta iz Canvas aplikacije |
| `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` | POTVRĐENO | Dodela delovodnog broja iz Power Apps procesa |
| Email intake flow | POTVRĐENO konceptualno | Čita shared mailbox i zavodi dokumente iz emailova sa attachmentom |
| Export flow | POTVRĐENO konceptualno | Export šifarnika i konfiguracije iz `AppConfig` liste |

Važna arhitektonska odluka:

- delovodni broj se ne dodeljuje direktno iz Canvas aplikacije
- postoji poseban flow zbog problema konkurentnog zavođenja dokumenata
- razlog je sprečavanje situacije da dva korisnika istovremeno dobiju isti delovodni broj

Nepoznato:

- trigger-i svih flow-ova
- kompletna logika akcija
- connection reference nazivi
- service account model
- error handling
- retry policy
- concurrency settings
- child flow struktura
- scope struktura
- logging model

---

### 4.3 SharePoint Online

SharePoint Online je primarni data layer i storage layer.

Poznata SharePoint lokacija:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

REST API base:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

Identifikovane liste i biblioteke:

| Naziv | Tip | Namena |
|---|---|---|
| `AppConfig` | SharePoint lista | Centralna konfiguracija, šifarnici i JSON podešavanja |
| `RezervisaniBrojevi` | SharePoint lista | Rezervacija / kontrola delovodnih brojeva |
| `Shared Documents` | Document library | Glavna biblioteka dokumenata |
| `EmailDocuments` | Document library | Dokumenti iz email intake procesa |
| `Exports` | Document library | Export fajlovi / izvoz konfiguracije i šifarnika |

---

## 5. Trenutni funkcionalni tokovi

### 5.1 Ručno zavođenje dokumenta iz Canvas aplikacije

Poznat tok na visokom nivou:

```text
Korisnik otvara Canvas aplikaciju
   |
   v
Korisnik ide na ekran za zavođenje dokumenta
   |
   v
Korisnik unosi podatke i dodaje dokument
   |
   v
Canvas aplikacija poziva Power Automate flow `Upload Doc`
   |
   v
Flow obrađuje upload dokumenta
   |
   v
Poseban flow dodeljuje delovodni broj
   |
   v
Dokument i metadata se upisuju u SharePoint
```

Ključna napomena:

Dodela delovodnog broja mora ostati backend proces. Canvas aplikacija ne sme biti izvor istine za finalni delovodni broj.

---

### 5.2 Dodela delovodnog broja

Potvrđeni flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Namena:

- dodela delovodnog broja novom dokumentu
- rešavanje problema race condition-a
- sprečavanje duplih delovodnih brojeva kada više korisnika istovremeno zavodi dokumenta

Trenutni poznati enterprise zahtev:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Arhitektonska implikacija:

- delovodni broj mora biti generisan server-side
- mora postojati centralizovana kontrola broja
- finalni broj mora imati unique constraint
- mora postojati retry logika
- mora postojati audit log pokušaja i grešaka

---

### 5.3 Email intake proces

Potvrđeno:

- postoji posebna funkcionalnost za email dokumente
- Power Automate čita shared mailbox
- dokumenti iz emailova sa attachmentom se zavode kroz sistem
- biblioteka `EmailDocuments` čuva podatke povezane sa email procesom

Identifikovana polja u biblioteci `EmailDocuments`:

| Internal name | Namena |
|---|---|
| `FileLeafRef` | Ime fajla |
| `Title` | Naslov |
| `_ExtendedDescription` | Opis |
| `PosiljalacEmail` | Email pošiljaoca |
| `Posiljalac` | Naziv / ime pošiljaoca |
| `MediaServiceImageTags` | Managed metadata / image tags |
| `ContentType` | Content type |

Nepoznato:

- naziv flow-a
- da li flow obrađuje samo attachment-e ili i email body
- da li postoji validacija pošiljaoca
- da li se email dokument odmah zavodi ili ide u međukorak
- kako se rešavaju greške pri obradi mailova
- da li postoji deduplikacija email attachment-a

---

### 5.4 Export proces

Potvrđeno:

- export služi za izvoz šifarnika i konfiguracije iz `AppConfig`
- biblioteka `Exports` služi za export fajlove

Identifikovana polja u biblioteci `Exports`:

| Internal name | Namena |
|---|---|
| `FileLeafRef` | Ime fajla |
| `Title` | Naslov |
| `_ExtendedDescription` | Opis |
| `ContentType` | Content type |

Nepoznato:

- naziv export flow-a
- format exporta
- da li se export generiše ručno ili automatski
- da li export sadrži sve AppConfig zapise ili samo deo
- da li postoji verzionisanje exporta
- da li postoji import / restore proces iz exporta

---

## 6. AppConfig kao konfiguracioni sloj

`AppConfig` je centralna lista za:

- šifarnike
- konfiguraciju aplikacije
- JSON podešavanja
- verovatno i definicije kolona / prikaza

Identifikovana polja:

| Internal name | Display name | Tip |
|---|---|---|
| `Title` | Title | Single line of text |
| `Config` | Config | Multiple lines of text |
| `ColumnHeader` | ColumnHeader | Multiple lines of text |
| `ID` | ID | Counter |
| `Created` | Created | Date and Time |
| `Modified` | Modified | Date and Time |
| `Author` | Created By | Person or Group |
| `Editor` | Modified By | Person or Group |

Arhitektonska vrednost:

- smanjuje potrebu za hardkodiranjem šifarnika u Canvas aplikaciji
- omogućava centralnu promenu podešavanja
- omogućava export konfiguracije
- može biti osnova za višejezičnost i dinamički UI

Rizik:

Ako Canvas aplikacija zavisi od JSON konfiguracije iz `AppConfig`, mora postojati:

- validacija JSON strukture
- fallback vrednosti
- verzionisanje konfiguracije
- kontrola ko sme da menja konfiguraciju
- audit promene konfiguracije

---

## 7. RezervisaniBrojevi kao deo numbering arhitekture

Lista `RezervisaniBrojevi` je posebno važna zbog delovodnih brojeva.

Identifikovana polja:

| Internal name | Display name | Tip |
|---|---|---|
| `Title` | Title | Single line of text |
| `RezervisaniBroj` | Rezervisani broj | Number |
| `DatumRezervacije` | DatumRezervacije | Date and Time |
| `ID` | ID | Counter |
| `Created` | Created | Date and Time |
| `Modified` | Modified | Date and Time |
| `Author` | Created By | Person or Group |
| `Editor` | Modified By | Person or Group |

Zaključak:

Lista najverovatnije učestvuje u procesu rezervacije ili kontrole brojeva.

Status:

```text
PRETPOSTAVKA
```

Razlog:

Dostavljena metadata potvrđuje strukturu liste, ali nije dostavljena kompletna logika flow-a koji koristi ovu listu.

---

## 8. Glavna biblioteka dokumenata — Shared Documents

`Shared Documents` je glavna biblioteka dokumenata.

Identifikovana važna polja:

| Internal name | Display name | Tip |
|---|---|---|
| `FileLeafRef` | Name | File |
| `Title` | Title | Single line of text |
| `_ExtendedDescription` | Description | Multiple lines of text |
| `DelovodniBroj` | DelovodniBroj | Single line of text |
| `EdokumentID` | EdokumentID | Single line of text |
| `Edokument` | Edokument | Yes/No |
| `Attachment` | Attachment | Yes/No |
| `MediaServiceImageTags` | Image Tags | Managed Metadata |
| `ContentType` | Content Type | Computed |

Zaključak:

Ova biblioteka čuva dokumente koji su povezani sa delovodnim brojem i statusom elektronskog dokumenta.

Rizik:

Polje `DelovodniBroj` nije potvrđeno kao unique u dostavljenoj metadata strukturi.

Preporuka:

U novoj verziji finalni delovodni broj mora imati jedinstvenu kontrolu na backend nivou i/ili kroz posebnu listu / tabelu sa unique constraint pravilom.

---

## 9. Trenutni poznati arhitektonski rizici

### 9.1 Race condition kod delovodnog broja

Najveći rizik je da dva korisnika istovremeno započnu zavođenje i dobiju isti broj.

Trenutno je potvrđeno da postoji poseban flow za ovu namenu:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Međutim, bez kompletne logike flow-a nije moguće potvrditi da je zaštita potpuno enterprise-safe.

Potrebno proveriti:

- da li flow koristi concurrency control
- da li koristi SharePoint unique constraint
- da li koristi ETag / optimistic locking
- da li ima retry logiku
- da li upisuje audit log
- da li vraća kontrolisanu grešku Canvas aplikaciji

---

### 9.2 Previše poslovne logike u Canvas aplikaciji

Ako se značajna poslovna logika nalazi u Canvas aplikaciji, rizici su:

- teže održavanje
- teže testiranje
- veći rizik od race condition problema
- veća zavisnost od lokalnih kolekcija
- veća mogućnost različitog ponašanja kod korisnika

Preporuka:

Canvas aplikacija treba da bude UI sloj, dok kritične operacije treba da budu u backend flow-ovima ili budućem servisnom sloju.

---

### 9.3 SharePoint kao enterprise backend

SharePoint Online može biti prihvatljiv storage layer, ali kod enterprise pisarnice mora se posebno kontrolisati:

- delegacija
- indexing
- list view threshold
- item-level permissions
- unique values
- throttling
- flow timeout
- retry logika
- audit
- backup i restore
- lifecycle dokumenata

---

### 9.4 AppConfig kao centralna konfiguracija

Ako je `AppConfig` kritičan za rad aplikacije, mora imati strogu kontrolu.

Rizici:

- pogrešan JSON može oboriti deo aplikacije
- nevalidna konfiguracija može sakriti ili promeniti poslovna pravila
- bez verzionisanja je teško vratiti prethodno stanje
- bez permission modela korisnici mogu slučajno promeniti konfiguraciju

---

## 10. Trenutna arhitektura — sažetak

```text
Power Apps Canvas App
- početni ekran: scrHome
- ekran za zavođenje dokumenta
- koristi AppConfig
- poziva flow-ove
- ne sme biti izvor istine za delovodni broj

Power Automate
- Upload Doc flow
- CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
- email intake flow
- export flow
- backend sloj za kritične operacije

SharePoint Online
- AppConfig
- RezervisaniBrojevi
- Shared Documents
- EmailDocuments
- Exports

Najveći enterprise zahtev
- više korisnika istovremeno zavodi dokumente
- delovodni broj nikada ne sme biti dupliran
```

---

## 11. Nepoznato / potrebno dopuniti

| Pitanje | Status |
|---|---|
| Kompletan export Canvas aplikacije | NEPOZNATO |
| Lista svih ekrana | DELIMIČNO POZNATO |
| Kompletna Power Apps formula logika | NEPOZNATO |
| Svi Power Automate flow-ovi | DELIMIČNO POZNATO |
| Detaljna logika `Upload Doc` flow-a | NEPOZNATO |
| Detaljna logika `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` flow-a | NEPOZNATO |
| Da li `DelovodniBroj` ima unique constraint | NEPOZNATO |
| Da li `RezervisaniBrojevi` koristi atomic/locking mehanizam | NEPOZNATO |
| Da li postoji audit lista za greške | NEPOZNATO |
| Da li postoji service account | NEPOZNATO |
| Da li korisnici imaju read-only prava nad SharePoint-om | NEPOZNATO za ovo konkretno okruženje |
| Da li Power Automate radi sa elevated permissions | NEPOZNATO |
| Da li postoji backup / restore konfiguracije | DELIMIČNO POZNATO kroz export, ali nije potvrđen restore |

---

## 12. Zaključak

Trenutna arhitektura DocCentral v6.0 je zasnovana na klasičnom Power Platform obrascu:

```text
Canvas App + Power Automate + SharePoint Online
```

Najvažnija pozitivna odluka je izdvajanje dodele delovodnog broja u poseban Power Automate flow, jer se time kritična logika pomera iz Canvas aplikacije ka backend procesu.

Najvažniji rizik i dalje ostaje dokazivanje da je dodela delovodnog broja zaista concurrency-safe.

Za novu enterprise verziju, trenutna arhitektura treba da evoluira ka jasnijem razdvajanju:

- UI sloj
- business service sloj
- numbering service
- audit/logging sloj
- konfiguracioni sloj
- dokument storage sloj
- security/permission sloj

---

## 13. Preporučeni sledeći dokumenti

Povezani dokumenti u repozitorijumu:

```text
docs/02-current-architecture.md
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
architecture/target-architecture.md
architecture/numbering-sequence.md
data-model/appconfig.md
data-model/rezervisani-brojevi.md
```
