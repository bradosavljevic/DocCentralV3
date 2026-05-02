# 12. Cloud Code development brief

## 1. Svrha dokumenta

Ovaj dokument predstavlja razvojni brief za novu enterprise verziju rešenja **DocCentral**.

Namenjen je za:

- Cloud Code
- Claude Code
- AI-assisted development
- GitHub dokumentaciju
- planiranje nove verzije Power Platform rešenja
- tehnički onboarding developera / arhitekte

Brief opisuje šta nova verzija treba da napravi, koja pravila mora da poštuje i koje tehničke odluke su obavezne.

---

## 2. Kontekst rešenja

DocCentral je Power Platform rešenje za elektronsku pisarnicu i centralnu evidenciju dokumenata.

Postojeća verzija koristi:

- Power Apps Canvas App
- Power Automate flow-ove
- SharePoint Online liste
- SharePoint document libraries
- `AppConfig` listu za konfiguracije, šifarnike i JSON podešavanja
- `RezervisaniBrojevi` listu za rezervaciju / kontrolu delovodnih brojeva
- `Shared Documents` biblioteku za glavne dokumente
- `EmailDocuments` biblioteku za dokumente pristigle emailom
- `Exports` biblioteku za export konfiguracija i šifarnika

Najvažniji zahtev nove verzije:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

---

## 3. Poznate komponente postojeće aplikacije

### 3.1 Power Apps

Poznato:

- aplikacija je Canvas App
- početni ekran je `scrHome`
- postoji posebna funkcionalnost za zavođenje dokumenta
- pregled dokumenata se trenutno radi direktno u SharePoint listi
- Canvas App ne sme biti izvor istine za finalni delovodni broj

### 3.2 Power Automate

Poznati flow-ovi / procesi:

| Naziv / opis | Namena |
|---|---|
| `Upload Doc` | Upload dokumenta preko Power Automate |
| `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` | Dodela / dodavanje delovodnog broja novom dokumentu iz Power Apps aplikacije |
| Email intake flow | Čita shared mailbox i zavodi dokumente iz mailova sa attachment-ima |
| Export flow | Export šifarnika i konfiguracije iz `AppConfig` |

Napomena:

```text
Flow za delovodni broj postoji zbog race condition problema koji se javlja kada dva korisnika istovremeno zavode dokument.
```

### 3.3 SharePoint

Poznate liste / biblioteke:

| Naziv | Tip | Namena |
|---|---|---|
| `AppConfig` | SharePoint lista | Centralna konfiguracija, šifarnici, JSON podešavanja |
| `RezervisaniBrojevi` | SharePoint lista | Rezervacija / kontrola delovodnih brojeva |
| `Shared Documents` | Document library | Glavna biblioteka dokumenata |
| `EmailDocuments` | Document library | Dokumenti iz email procesa |
| `Exports` | Document library | Export konfiguracija i šifarnika |

---

## 4. Glavni cilj nove verzije

Nova verzija treba da bude enterprise-ready rešenje za pisarnicu koje ima:

1. sigurnu server-side dodelu delovodnog broja
2. zaštitu od duplih delovodnih brojeva
3. stabilan upload dokumenata
4. audit log svih važnih akcija
5. centralizovanu konfiguraciju
6. jasnu podelu UI / backend / data sloja
7. role-based security
8. bolji admin deo
9. bolju dijagnostiku grešaka
10. GitHub dokumentaciju kao osnovu za dalji razvoj

---

## 5. Obavezna arhitekturna pravila

### 5.1 Power Apps je UI sloj

Canvas App treba da radi samo:

- prikaz ekrana
- unos podataka
- osnovnu validaciju
- pozivanje Power Automate flow-ova
- prikaz rezultata
- prikaz grešaka
- navigaciju

Canvas App ne sme samostalno da:

- generiše konačni delovodni broj
- garantuje jedinstvenost delovodnog broja
- koristi `Max(...) + 1` kao finalni mehanizam za broj
- direktno upisuje kritične podatke bez backend validacije
- zaobilazi flow za zavođenje dokumenta

### 5.2 Power Automate je poslovni sloj

Power Automate treba da radi:

- validaciju zahteva
- rezervaciju delovodnog broja
- upis dokumenta
- upload fajla
- update metapodataka
- audit log
- retry logiku
- kontrolu grešaka
- response prema Canvas aplikaciji

### 5.3 SharePoint je data i storage sloj

SharePoint treba da čuva:

- dokumente
- metapodatke
- konfiguracije
- šifarnike
- rezervisane brojeve
- logove
- export fajlove

---

## 6. Kritičan zahtev: jedinstven delovodni broj

### 6.1 Problem

Kada dva korisnika istovremeno kliknu na zavođenje dokumenta, sistem ne sme da im dodeli isti delovodni broj.

Ovo je najvažniji enterprise zahtev.

### 6.2 Obavezno rešenje

Implementirati centralizovan backend mehanizam:

```text
Canvas App -> Power Automate -> RezervisaniBrojevi -> Shared Documents
```

Finalni delovodni broj mora nastati u flow-u, ne u Canvas aplikaciji.

### 6.3 Obavezne zaštite

Implementirati:

- atomic ili optimistic locking pristup
- unique key
- retry logiku
- audit log
- standardizovan error response
- correlation ID
- status rezervacije
- kontrolu neuspele rezervacije
- kontrolu napuštene rezervacije

### 6.4 Predlog unique key formata

U `RezervisaniBrojevi` dodati polje:

```text
UniqueReservationKey
```

Primer vrednosti:

```text
2026|OS|123
```

Na ovom polju uključiti unique constraint.

---

## 7. Predloženi data model

### 7.1 AppConfig

Koristi se za:

- šifarnike
- konfiguracije
- JSON podešavanja
- export konfiguracija

Postojeća poznata polja:

| Internal name | Tip |
|---|---|
| `Title` | Text |
| `Config` | Multiple lines of text |
| `ColumnHeader` | Multiple lines of text |
| `ID` | Counter |
| `Created` | Date and Time |
| `Modified` | Date and Time |

Predložena dodatna polja:

| Internal name | Tip |
|---|---|
| `IsActive` | Yes/No |
| `ConfigVersion` | Text |
| `Environment` | Choice |
| `Description` | Multiple lines of text |
| `LastExportedAt` | Date and Time |
| `LastExportedBy` | Text |

---

### 7.2 RezervisaniBrojevi

Postojeća poznata polja:

| Internal name | Tip |
|---|---|
| `Title` | Text |
| `RezervisaniBroj` | Number |
| `DatumRezervacije` | Date and Time |
| `ID` | Counter |
| `Created` | Date and Time |
| `Modified` | Date and Time |

Predložena dodatna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Godina` | Number | Godina rezervacije |
| `TipDokumenta` | Text/Choice | Tip dokumenta |
| `Prefix` | Text | Prefix broja |
| `FullDelovodniBroj` | Text | Kompletan delovodni broj |
| `UniqueReservationKey` | Text | Jedinstveni ključ |
| `ReservationStatus` | Choice | Reserved, Used, Failed, Cancelled, Expired |
| `ReservedByEmail` | Text | Korisnik koji je pokrenuo akciju |
| `ReservedForItemId` | Number | ID dokumenta |
| `CorrelationId` | Text | ID procesa |
| `RequestPayload` | Multiple lines of text | Ulazni JSON |
| `ResponsePayload` | Multiple lines of text | Izlazni JSON |
| `ErrorDetails` | Multiple lines of text | Greška |

---

### 7.3 Shared Documents

Postojeća poznata polja:

| Internal name | Tip |
|---|---|
| `FileLeafRef` | File |
| `Title` | Text |
| `_ExtendedDescription` | Multiple lines of text |
| `DelovodniBroj` | Text |
| `EdokumentID` | Text |
| `Edokument` | Yes/No |
| `Attachment` | Yes/No |
| `MediaServiceImageTags` | Managed Metadata |
| `ContentType` | Computed |

Predložena dodatna polja:

| Internal name | Tip |
|---|---|
| `DocumentStatus` | Choice |
| `RegistrationDate` | Date and Time |
| `RegisteredByEmail` | Text |
| `SourceType` | Choice |
| `SourceReferenceId` | Text |
| `CorrelationId` | Text |
| `ProcessingError` | Multiple lines of text |

---

### 7.4 EmailDocuments

Postojeća poznata polja:

| Internal name | Tip |
|---|---|
| `FileLeafRef` | File |
| `Title` | Text |
| `_ExtendedDescription` | Multiple lines of text |
| `PosiljalacEmail` | Text |
| `Posiljalac` | Text |
| `MediaServiceImageTags` | Managed Metadata |
| `ContentType` | Computed |

Predložena dodatna polja:

| Internal name | Tip |
|---|---|
| `EmailMessageId` | Text |
| `EmailSubject` | Text |
| `ReceivedDateTime` | Date and Time |
| `AttachmentName` | Text |
| `AttachmentHash` | Text |
| `ProcessingStatus` | Choice |
| `ProcessingError` | Multiple lines of text |
| `DelovodniBroj` | Text |
| `CorrelationId` | Text |

---

### 7.5 Exports

Postojeća poznata polja:

| Internal name | Tip |
|---|---|
| `FileLeafRef` | File |
| `Title` | Text |
| `_ExtendedDescription` | Multiple lines of text |
| `ContentType` | Computed |

Predložena dodatna polja:

| Internal name | Tip |
|---|---|
| `ExportType` | Choice |
| `ExportedByEmail` | Text |
| `ExportedAt` | Date and Time |
| `ConfigVersion` | Text |
| `Environment` | Choice |
| `CorrelationId` | Text |
| `ExportStatus` | Choice |
| `ExportError` | Multiple lines of text |

---

## 8. Predloženi Power Automate flow-ovi

### 8.1 `PA_RegisterDocument`

Novi ili refaktorisani glavni flow za zavođenje dokumenta.

Odgovornosti:

- prima zahtev iz Canvas App
- validira ulaz
- poziva logiku za rezervaciju broja
- kreira ili ažurira dokument
- upisuje audit
- vraća standardizovan response

### 8.2 `PA_ReserveWorkbookNumber`

Poseban child flow ili glavni flow za dodelu broja.

Može biti refaktorisani naslednik postojećeg flow-a:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Odgovornosti:

- pronalazi sledeći broj
- kreira unique reservation key
- pokušava upis u `RezervisaniBrojevi`
- ako dođe do konflikta, pokušava sledeći broj
- vraća rezervisani broj

### 8.3 `PA_UploadDoc`

Refaktorisani naslednik postojećeg flow-a:

```text
Upload Doc
```

Odgovornosti:

- prima fajl iz Canvas App
- validira fajl
- kreira fajl u biblioteci
- upisuje metapodatke
- vraća URL, item ID i status

### 8.4 `PA_EmailIntake`

Flow za shared mailbox.

Odgovornosti:

- čita shared mailbox
- proverava email attachment-e
- sprečava duplikate
- upisuje fajlove u `EmailDocuments`
- pokreće proces zavođenja ako je definisano pravilima
- loguje greške

### 8.5 `PA_ExportAppConfig`

Flow za export `AppConfig`.

Odgovornosti:

- čita aktivne konfiguracije
- validira JSON
- generiše export fajl
- upisuje fajl u `Exports`
- vraća link ka exportu

### 8.6 `PA_WriteAuditLog`

Centralni flow za audit log.

Odgovornosti:

- prima audit payload
- upisuje log u SharePoint listu ili Dataverse
- ne prekida glavni proces ako logovanje ne uspe, osim za kritične audit događaje

---

## 9. Standardizovani JSON response

Svaki flow koji se poziva iz Canvas aplikacije treba da vraća isti osnovni format.

### 9.1 Uspešan response

```json
{
  "success": true,
  "status": "Success",
  "message": "Akcija je uspešno izvršena.",
  "data": {
    "delovodniBroj": "Os.Del.Br. 123/2026",
    "documentItemId": 1001,
    "fileUrl": "https://..."
  },
  "correlationId": "GUID",
  "error": null
}
```

### 9.2 Neuspešan response

```json
{
  "success": false,
  "status": "Failed",
  "message": "Akcija nije izvršena.",
  "data": null,
  "correlationId": "GUID",
  "error": {
    "code": "NUMBER_RESERVATION_FAILED",
    "details": "Technical error details"
  }
}
```

---

## 10. Predlog Canvas App ekrana

### 10.1 Obavezni ekrani

| Ekran | Namena |
|---|---|
| `scrHome` | Početni ekran |
| `scrRegisterDocument` | Zavođenje dokumenta |
| `scrUploadDocument` | Upload dokumenta |
| `scrEmailDocuments` | Pregled email dokumenata |
| `scrAppConfigAdmin` | Administracija konfiguracije |
| `scrExportConfig` | Export konfiguracija |
| `scrLogs` | Pregled logova |
| `scrErrorDetails` | Detalji grešaka |

### 10.2 Pravila za Canvas App

Aplikacija mora:

- biti responzivna
- podržati srpski i engleski jezik
- imati centralizovane komponente za poruke
- koristiti standardizovane flow response objekte
- imati loading indikatore
- imati kontrolu grešaka
- imati role-based navigaciju
- ne sme sadržati kritičnu backend logiku za delovodni broj

---

## 11. Security pravila

### 11.1 Permission model

Preporučeni model:

- korisnici imaju read-only pristup SharePoint podacima
- Power Automate flow-ovi rade upise
- flow-ovi koriste kontrolisan service account
- stvarni korisnik se uvek prosleđuje i loguje
- admin funkcije su dostupne samo administratorima

### 11.2 Role

Predložene role:

| Rola | Namena |
|---|---|
| `DocCentral.Reader` | Pregled |
| `DocCentral.Clerk` | Zavođenje dokumenta |
| `DocCentral.Manager` | Nadzor |
| `DocCentral.Admin` | Administracija |
| `DocCentral.ServiceAccount` | Tehnički nalog za flow-ove |

---

## 12. Logging i audit

### 12.1 Obavezna logika

Svaka kritična akcija mora imati:

- correlation ID
- korisnika koji je pokrenuo akciju
- naziv flow-a
- vreme početka
- vreme završetka
- status
- request payload
- response payload
- error payload

### 12.2 Predložena lista

```text
FlowExecutionLog
```

Predložena polja:

| Internal name | Tip |
|---|---|
| `Title` | Text |
| `CorrelationId` | Text |
| `FlowName` | Text |
| `FlowRunId` | Text |
| `ActionType` | Text |
| `Status` | Choice |
| `StartedAt` | Date and Time |
| `FinishedAt` | Date and Time |
| `DurationMs` | Number |
| `InitiatedByEmail` | Text |
| `TargetItemId` | Number |
| `RequestJson` | Multiple lines of text |
| `ResponseJson` | Multiple lines of text |
| `ErrorJson` | Multiple lines of text |

---

## 13. Cloud Code zadatak

Cloud Code treba da generiše tehničku osnovu za novu verziju rešenja kroz GitHub projekat.

### 13.1 Obavezno generisati dokumentaciju

Struktura:

```text
doccentral-vNext/
├── README.md
├── docs/
│   ├── business-overview.md
│   ├── architecture.md
│   ├── data-model.md
│   ├── power-apps-design.md
│   ├── power-automate-design.md
│   ├── concurrency-numbering.md
│   ├── security-model.md
│   ├── logging-audit.md
│   ├── deployment.md
│   └── open-questions.md
├── power-platform/
│   ├── canvas-app/
│   │   ├── screens.md
│   │   ├── components.md
│   │   └── formulas.md
│   ├── flows/
│   │   ├── PA_RegisterDocument.md
│   │   ├── PA_ReserveWorkbookNumber.md
│   │   ├── PA_UploadDoc.md
│   │   ├── PA_EmailIntake.md
│   │   └── PA_ExportAppConfig.md
│   └── sharepoint/
│       ├── lists.md
│       ├── libraries.md
│       └── indexes.md
├── prompts/
│   └── cloud-code-master-prompt.md
└── backlog/
    ├── enterprise-backlog.md
    ├── technical-debt.md
    └── risks.md
```

### 13.2 Obavezno poštovati

Cloud Code mora poštovati sledeća pravila:

- ne izmišljati postojeće ekrane osim ako su označeni kao predlog
- konkretno koristiti poznate nazive:
  - `scrHome`
  - `AppConfig`
  - `RezervisaniBrojevi`
  - `Shared Documents`
  - `EmailDocuments`
  - `Exports`
  - `Upload Doc`
  - `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- sve nepoznato označiti kao `NEPOZNATO`
- jasno razdvajati:
  - činjenice
  - pretpostavke
  - predloge
- najkritičniji zahtev je sprečavanje duplog delovodnog broja
- finalni delovodni broj ne sme se generisati samo u Canvas aplikaciji

---

## 14. Master prompt za Cloud Code

```text
Radi kao senior Power Platform arhitekta i enterprise solution architect.

Kontekst:
Pravimo novu enterprise verziju rešenja DocCentral, aplikacije za elektronsku pisarnicu i centralnu evidenciju dokumenata.

Postojeća verzija koristi:
- Power Apps Canvas App
- Power Automate flow-ove
- SharePoint Online liste i biblioteke
- AppConfig listu za šifarnike, konfiguraciju i JSON podešavanja
- RezervisaniBrojevi listu za rezervaciju / kontrolu delovodnih brojeva
- Shared Documents biblioteku za glavne dokumente
- EmailDocuments biblioteku za dokumente iz email procesa
- Exports biblioteku za export šifarnika i konfiguracije

Poznato:
- početni ekran Canvas aplikacije je scrHome
- upload dokumenta ide preko Power Automate flow-a Upload Doc
- dodela delovodnog broja ide preko posebnog flow-a CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
- taj flow postoji zbog race condition problema kada više korisnika istovremeno zavodi dokumenta
- email proces koristi shared mailbox i attachment-e
- export proces izvozi šifarnike i konfiguracije iz AppConfig liste

Najvažniji enterprise zahtev:
Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Obavezna pravila:
- Power Apps je UI sloj
- Power Automate je backend/business sloj
- SharePoint je data i document storage sloj
- finalni delovodni broj ne sme da se generiše samo u Canvas aplikaciji
- mora postojati server-side concurrency-safe mehanizam
- mora postojati unique constraint ili unique key
- mora postojati retry logika
- mora postojati audit log
- mora postojati correlation ID
- sve Create/Edit/Delete operacije treba da idu preko Power Automate
- korisnici treba da imaju read-only prava nad SharePoint podacima gde je moguće
- service account izvršava upise kroz flow-ove
- stvarni korisnik mora biti upisan u audit log

Zadatak:
Napravi GitHub-ready tehničku osnovu za novu verziju rešenja.

Generiši:
1. README.md
2. business overview
3. current architecture
4. target architecture
5. SharePoint data model
6. Power Apps design
7. Power Automate design
8. concurrency-safe numbering design
9. security model
10. logging and audit model
11. deployment notes
12. open questions
13. backlog
14. risk register

U dokumentaciji jasno razdvoji:
- činjenice
- pretpostavke
- predloge
- nepoznato

Ne izmišljaj postojeće elemente.
Ako nešto nije poznato, napiši NEPOZNATO.
Kada predlažeš novu komponentu, jasno označi kao PREDLOG.
```

---

## 15. Otvorena pitanja za razvoj

1. Da li nova verzija ostaje na SharePoint Online data modelu ili se planira Dataverse?
2. Da li korisnici u produkciji treba da imaju potpuno read-only prava nad listama?
3. Da li service account već postoji?
4. Da li postoji postojeća audit lista?
5. Da li postoji postojeća log lista?
6. Da li `RezervisaniBrojevi` trenutno ima unique constraint?
7. Da li `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` koristi retry logiku?
8. Da li `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` koristi ETag / optimistic concurrency?
9. Da li `Upload Doc` vraća JSON response?
10. Da li postoje Dev/Test/Prod okruženja?
11. Da li se `AppConfig` razlikuje po okruženjima?
12. Da li email intake odmah zavodi dokument ili prvo pravi staging dokument?
13. Da li je potrebno odobravanje pre finalnog zavođenja?
14. Da li postoje posebne knjige / tipovi delovodnika?
15. Da li delovodni broj zavisi od godine, tipa dokumenta, organizacione jedinice ili kombinacije?

---

## 16. Zaključak

Nova verzija DocCentral rešenja treba da bude projektovana oko jednog ključnog principa:

```text
Canvas App ne sme biti izvor istine za kritične poslovne odluke.
```

Za pisarnicu je najkritičnija odluka dodela delovodnog broja.

Zato buduća verzija mora imati backend-controlled, concurrency-safe mehanizam za dodelu broja, sa audit logom, retry logikom i unique constraint zaštitom.

Ovaj dokument je polazni brief za razvoj nove verzije kroz Cloud Code / Claude Code / AI-assisted development.
