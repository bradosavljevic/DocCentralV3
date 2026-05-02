# Open Questions

## 1. Svrha dokumenta

Ovaj dokument sadrži otvorena pitanja za DocCentral v6.0 i novu enterprise verziju rešenja.

Cilj dokumenta je da se pre razvoja zatvore nepoznanice koje utiču na:

- arhitekturu
- SharePoint data model
- Power Apps Canvas App
- Power Automate flow-ove
- sigurnost
- concurrency-safe dodelu delovodnog broja
- email intake proces
- export konfiguracije
- ALM i deployment

---

## 2. Status informacija

U ovom dokumentu koriste se sledeće oznake:

| Oznaka | Značenje |
|---|---|
| POTVRĐENO | Informacija je potvrđena od strane korisnika ili iz dostavljenih metadata podataka |
| PRETPOSTAVKA | Zaključak je izveden iz naziva lista, polja ili procesa |
| NEPOZNATO | Nema dovoljno podataka za pouzdan zaključak |
| PITANJE | Potrebna je dodatna potvrda |

---

## 3. Potvrđene informacije

### 3.1 Tip aplikacije

Status:

```text
POTVRĐENO
```

Aplikacija je:

```text
Power Apps Canvas App
```

Nije SharePoint customized form.

---

### 3.2 Početni ekran

Status:

```text
POTVRĐENO
```

Početni ekran aplikacije je:

```text
scrHome
```

---

### 3.3 Zavođenje dokumenta

Status:

```text
POTVRĐENO
```

Postoji poseban ekran / funkcionalnost za zavođenje dokumenta.

---

### 3.4 Pregled dokumenata

Status:

```text
POTVRĐENO
```

Pregled dokumenata se ne radi kroz poseban Canvas App ekran.

Dokumenti se pregledaju direktno u SharePoint listi / biblioteci.

---

### 3.5 Email dokumenti

Status:

```text
POTVRĐENO
```

Email dokumenti su posebna funkcionalnost.

Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.

---

### 3.6 Export

Status:

```text
POTVRĐENO
```

Export se odnosi na:

- šifarnike
- konfiguraciju iz `AppConfig` liste

---

### 3.7 Administracija i konfiguracija

Status:

```text
POTVRĐENO
```

Svi šifarnici i konfiguracioni JSON podaci nalaze se u listi:

```text
AppConfig
```

---

### 3.8 Upload dokumenta

Status:

```text
POTVRĐENO
```

Upload dokumenta ide preko Power Automate flow-a:

```text
Upload Doc
```

---

### 3.9 Poseban flow za dodelu delovodnog broja

Status:

```text
POTVRĐENO
```

Zbog race condition problema postoji poseban flow za dodelu delovodnog broja.

Naziv flow-a:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

---

### 3.10 Kritičan concurrency zahtev

Status:

```text
POTVRĐENO
```

Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovo je P0 zahtev.

---

## 4. Otvorena pitanja po oblastima

---

# 4.1 Canvas App

## Q-APP-001 — Kompletan spisak ekrana

Status:

```text
NEPOZNATO
```

Poznato:

```text
scrHome
```

Potrebno potvrditi:

- tačan naziv ekrana za zavođenje dokumenta
- da li postoje pomoćni ekrani
- da li postoje admin ekrani
- da li postoje modal / popup kontejneri
- da li postoje skriveni tehnički ekrani

Pitanje:

```text
Koji su tačni nazivi svih ekrana u Canvas App aplikaciji?
```

---

## Q-APP-002 — Naziv ekrana za zavođenje dokumenta

Status:

```text
NEPOZNATO
```

Poznato:

- funkcionalnost postoji
- nije dostavljen tačan naziv ekrana

Pitanje:

```text
Kako se tačno zove ekran na kome korisnik zavodi dokument?
```

---

## Q-APP-003 — Kolekcije u aplikaciji

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- koje kolekcije se kreiraju na startu aplikacije
- koje kolekcije dolaze iz `AppConfig`
- koje kolekcije dolaze iz Power Automate flow-ova
- koje kolekcije služe za zavođenje dokumenta
- koje kolekcije se koriste za korisničke role i permissions

Pitanje:

```text
Koje su glavne kolekcije u aplikaciji i čemu služe?
```

---

## Q-APP-004 — Globalne varijable

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- globalne varijable za korisnika
- globalne varijable za role
- globalne varijable za konfiguraciju
- globalne varijable za formatiranje
- globalne varijable za flow response

Pitanje:

```text
Koje globalne varijable koristi aplikacija?
```

---

## Q-APP-005 — Direktni SharePoint upisi iz Canvas App

Status:

```text
NEPOZNATO
```

Poznato:

- upload ide preko flow-a `Upload Doc`
- numbering ide preko flow-a `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- u enterprise verziji kritični upisi treba da idu preko Power Automate

Potrebno potvrditi:

- da li Canvas App sada negde direktno koristi `Patch`
- koje liste se direktno patch-uju
- da li postoje direktni update/delete pozivi
- da li postoje direktni upisi u `AppConfig`
- da li postoje direktni upisi u `RezervisaniBrojevi`

Pitanje:

```text
Da li Canvas App trenutno direktno upisuje u neke SharePoint liste preko Patch funkcije?
```

---

# 4.2 Power Automate

## Q-FLOW-001 — Kompletan spisak flow-ova

Status:

```text
NEPOZNATO
```

Poznati flow-ovi:

```text
Upload Doc
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Potrebno potvrditi:

- svi flow-ovi koje poziva Canvas App
- svi scheduled flow-ovi
- svi flow-ovi za email intake
- svi flow-ovi za export
- svi child flow-ovi
- svi flow-ovi za administraciju

Pitanje:

```text
Koji su tačni nazivi svih Power Automate flow-ova u solution-u?
```

---

## Q-FLOW-002 — Ulazni JSON za `Upload Doc`

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- parametri koje Canvas App šalje
- da li se šalje file content
- da li se šalje file name
- da li se šalje metadata
- da li se šalje korisnik
- da li se šalje delovodni broj ili samo privremeni ID
- da li se šalje correlation ID

Pitanje:

```text
Koji je tačan input JSON schema za flow Upload Doc?
```

---

## Q-FLOW-003 — Izlazni JSON iz `Upload Doc`

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- da li flow vraća ID dokumenta
- da li flow vraća URL
- da li flow vraća status
- da li flow vraća grešku
- da li flow vraća correlation ID
- da li flow vraća finalni `DelovodniBroj`

Pitanje:

```text
Koji je tačan output koji Upload Doc vraća Canvas aplikaciji?
```

---

## Q-FLOW-004 — Logika flow-a `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`

Status:

```text
NEPOZNATO
```

Poznato:

- flow postoji
- koristi se zbog race condition problema
- služi za dodelu delovodnog broja

Potrebno potvrditi:

- trigger
- input parametri
- da li koristi `RezervisaniBrojevi`
- da li koristi unique constraint
- da li ima retry logiku
- da li ažurira dokument
- da li vraća broj Canvas aplikaciji
- šta radi ako rezervacija ne uspe

Pitanje:

```text
Možeš li poslati export ili screenshot ključnih koraka flow-a CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps?
```

---

## Q-FLOW-005 — Email intake flow

Status:

```text
NEPOZNATO
```

Poznato:

- Power Automate čita shared mailbox
- zavodi dokumente iz mailova sa attachment-om
- koristi se biblioteka `EmailDocuments`

Potrebno potvrditi:

- tačan naziv flow-a
- shared mailbox adresa
- da li obrađuje sve attachment-e ili samo određene tipove
- da li sprečava duplu obradu maila
- da li koristi isti numbering flow
- da li čuva `MessageId`
- da li čuva `InternetMessageId`
- da li čuva sender podatke

Pitanje:

```text
Kako se zove email intake flow i koja je njegova osnovna logika?
```

---

## Q-FLOW-006 — Export flow

Status:

```text
NEPOZNATO
```

Poznato:

- export služi za šifarnike i konfiguraciju iz `AppConfig`
- postoji biblioteka `Exports`

Potrebno potvrditi:

- tačan naziv flow-a
- format exporta
- da li export ide u JSON, Excel, CSV ili drugi format
- da li export čuva verziju
- da li exportuje sve konfiguracije ili samo izabrane
- da li ga pokreće korisnik ili schedule

Pitanje:

```text
Kako se zove flow za export AppConfig šifarnika i konfiguracije?
```

---

# 4.3 SharePoint liste i biblioteke

## Q-SP-001 — Da li postoji centralna registry lista?

Status:

```text
NEPOZNATO
```

Poznate biblioteke/liste:

- `AppConfig`
- `RezervisaniBrojevi`
- `Shared Documents`
- `EmailDocuments`
- `Exports`

Potrebno potvrditi:

- da li postoji posebna lista za centralnu evidenciju dokumenata
- da li se glavni registry vodi samo kroz dokument biblioteku
- da li postoji lista poput `Svi predmeti`

Pitanje:

```text
Da li u ovom DocCentral v6.0 solution-u postoji posebna centralna lista za sve zavedene dokumente/predmete?
```

---

## Q-SP-002 — Finalno mesto čuvanja delovodnog broja

Status:

```text
NEPOZNATO
```

Poznato:

- `Shared Documents` ima polje `DelovodniBroj`
- verovatno se tu čuva broj za dokumente

Potrebno potvrditi:

- da li se `DelovodniBroj` čuva samo u biblioteci
- da li se čuva i u posebnoj listi
- da li se čuva u `RezervisaniBrojevi`
- da li email dokumenti kasnije dobijaju broj

Pitanje:

```text
Gde je finalno i zvanično mesto čuvanja delovodnog broja?
```

---

## Q-SP-003 — Unique constraint na `DelovodniBroj`

Status:

```text
NEPOZNATO
```

Poznato:

- u dostavljenom XML-u `DelovodniBroj` nije označen kao unique
- `Indexed` je prikazan kao `false` u dostavljenom segmentu

Pitanje:

```text
Da li je na polju DelovodniBroj trenutno uključeno Enforce unique values?
```

---

## Q-SP-004 — Unique constraint na `RezervisaniBrojevi`

Status:

```text
NEPOZNATO
```

Poznato:

- lista `RezervisaniBrojevi` postoji
- polja su `Title`, `RezervisaniBroj`, `DatumRezervacije`
- `RezervisaniBroj` u XML-u nije označen kao unique

Pitanje:

```text
Da li lista RezervisaniBrojevi trenutno ima bilo koje polje sa Enforce unique values?
```

---

## Q-SP-005 — Značenje polja `Attachment`

Status:

```text
NEPOZNATO
```

Poznato:

U `Shared Documents` postoji boolean polje:

```text
Attachment
```

Pitanje:

```text
Šta tačno znači polje Attachment u biblioteci Shared Documents?
```

Moguće pretpostavke:

- označava da je dokument attachment iz emaila
- označava da dokument ima povezane priloge
- označava da je dokument sekundarni fajl
- označava nešto specifično za proces

Status:

```text
PRETPOSTAVKA
```

---

## Q-SP-006 — Značenje polja `Edokument` i `EdokumentID`

Status:

```text
NEPOZNATO
```

Poznata polja:

```text
Edokument
EdokumentID
```

Pitanje:

```text
Da li se Edokument odnosi na elektronski dokument iz eksternog sistema i šta je EdokumentID?
```

---

# 4.4 Security i permissions

## Q-SEC-001 — Trenutni permission model

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- ko ima read prava
- ko ima edit prava
- koji nalog izvršava flow-ove
- da li se koristi service account
- da li korisnici mogu direktno menjati biblioteke
- da li se koriste SharePoint grupe
- da li se koriste M365/Entra security grupe

Pitanje:

```text
Koji je trenutni permission model za korisnike, administratore i service account?
```

---

## Q-SEC-002 — Vlasnik flow-ova

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Ko je vlasnik kritičnih flow-ova, posebno Upload Doc i CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps?
```

---

## Q-SEC-003 — Prava nad `RezervisaniBrojevi`

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Da li obični korisnici imaju edit prava nad listom RezervisaniBrojevi?
```

Preporuka:

```text
Ne bi trebalo da imaju edit prava.
```

---

# 4.5 AppConfig

## Q-CONFIG-001 — Primer sadržaja `Config` polja

Status:

```text
NEPOZNATO
```

Poznato:

- `AppConfig` ima polje `Config`
- tip je Multiple lines of text
- korisnik je potvrdio da su šifarnici i config JSON u `AppConfig`

Pitanje:

```text
Možeš li poslati jedan ili više primera JSON sadržaja iz AppConfig.Config polja?
```

---

## Q-CONFIG-002 — Značenje `ColumnHeader`

Status:

```text
NEPOZNATO
```

Poznato:

- `AppConfig` ima polje `ColumnHeader`
- tip je Multiple lines of text

Pitanje:

```text
Šta tačno sadrži ColumnHeader polje?
```

Moguća pretpostavka:

```text
Sadrži konfiguraciju prikaza kolona ili mapping zaglavlja za export/import.
```

Status:

```text
PRETPOSTAVKA
```

---

## Q-CONFIG-003 — Aktivna konfiguracija

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Kako aplikacija zna koji AppConfig red je aktivan?
```

Moguće varijante:

- po `Title`
- po ID-u
- po nekom key-u u JSON-u
- učitava sve redove
- koristi hardkodovane nazive

---

# 4.6 ALM i GitHub

## Q-ALM-001 — Solution environment variables

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- da li solution koristi environment variables
- koji su nazivi
- koje vrednosti imaju u dev/test/prod okruženju
- da li se koriste za site URL, list names, library names

Pitanje:

```text
Da li Power Platform solution koristi environment variables?
```

---

## Q-ALM-002 — Connection references

Status:

```text
NEPOZNATO
```

Potrebno potvrditi:

- koje connection references postoje
- koji flow ih koristi
- koji konektori su standard
- da li postoji premium konektor
- da li su connection references stabilne pri importu

Pitanje:

```text
Koje connection references postoje u solution-u?
```

---

## Q-ALM-003 — Deployment proces

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Da li postoji odvojen Dev/Test/Prod deployment proces ili se radi direktno u produkcionom okruženju?
```

---

# 4.7 Monitoring i audit

## Q-AUDIT-001 — Postojeći audit log

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Da li trenutno postoji posebna lista za audit log ili flow execution log?
```

---

## Q-AUDIT-002 — Logovanje grešaka iz flow-ova

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Gde se trenutno čuvaju greške iz flow-ova?
```

Moguće varijante:

- samo u Power Automate run history
- u SharePoint listi
- u AppConfig
- ne čuvaju se posebno

---

## Q-AUDIT-003 — Correlation ID

Status:

```text
NEPOZNATO
```

Pitanje:

```text
Da li flow-ovi trenutno generišu correlation ID koji se vraća Canvas aplikaciji i upisuje u log?
```

---

## 5. Pitanja najvišeg prioriteta

Sledeća pitanja treba zatvoriti prva jer direktno utiču na enterprise arhitekturu:

1. Kako tačno radi `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`?
2. Da li `RezervisaniBrojevi` ima unique constraint?
3. Da li `DelovodniBroj` ima unique constraint?
4. Gde se finalno čuva delovodni broj?
5. Da li Canvas App direktno patch-uje SharePoint liste?
6. Koji nalog izvršava kritične flow-ove?
7. Da li postoji centralni audit log?
8. Koji je input/output format za `Upload Doc`?
9. Koji je naziv i logika email intake flow-a?
10. Da li postoji posebna centralna registry lista za dokumente?

---

## 6. Zaključak

Trenutno je potvrđena osnovna arhitektura:

```text
Canvas App + Power Automate + SharePoint Online
```

Takođe je potvrđeno da su ključne funkcionalnosti:

- ručno zavođenje dokumenata
- upload preko `Upload Doc` flow-a
- dodela broja preko posebnog flow-a
- email intake preko shared mailbox-a
- konfiguracija kroz `AppConfig`
- export šifarnika i konfiguracije

Najveća nepoznanica ostaje detaljna implementacija kritičnog procesa:

```text
concurrency-safe dodela delovodnog broja
```

Dok se to ne potvrdi, nova enterprise verzija mora se projektovati tako da broj uvek dodeljuje centralizovani backend proces sa unique constraint-om, retry logikom i audit logom.
