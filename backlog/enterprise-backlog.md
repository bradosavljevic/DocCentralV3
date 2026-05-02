# Enterprise Backlog

## 1. Svrha dokumenta

Ovaj dokument definiše razvojni backlog za novu enterprise verziju DocCentral rešenja.

Backlog je zasnovan na:

- dostavljenim SharePoint REST XML metadata podacima
- potvrđenim informacijama o Canvas App aplikaciji
- potvrđenim informacijama o Power Automate flow-ovima
- ključnom zahtevu za bezbedno generisanje delovodnog broja
- poznatim rizicima i tehničkom dugu postojećeg rešenja

Cilj dokumenta je da razvojni tim, Cloud Code / Claude Code ili drugi AI-assisted development alat može jasno da razume šta treba projektovati i razviti u sledećoj verziji.

---

## 2. Ključni enterprise cilj

Nova verzija DocCentral rešenja mora omogućiti:

```text
Više korisnika istovremeno zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

Ovaj zahtev je obavezan i ima najviši prioritet.

---

## 3. Prioriteti

Backlog koristi sledeće prioritete:

| Prioritet | Značenje |
|---|---|
| P0 | Kritično, mora biti rešeno pre produkcije |
| P1 | Visok prioritet, potrebno za stabilnu enterprise upotrebu |
| P2 | Srednji prioritet, važno za održavanje i skaliranje |
| P3 | Nice-to-have ili kasnija faza |

---

## 4. Epici

Identifikovani su sledeći razvojni epici:

1. Numbering & concurrency
2. Document upload orchestration
3. Email intake processing
4. AppConfig governance
5. Security & permissions
6. Audit & monitoring
7. Canvas App modernization
8. SharePoint data model hardening
9. ALM & deployment
10. Reporting & export

---

# Epic 1: Numbering & Concurrency

## E1-001 — Centralizovani servis za dodelu delovodnog broja

Prioritet:

```text
P0
```

Opis:

Finalna dodela delovodnog broja mora biti izvršena u server-side logici, preko Power Automate flow-a ili drugog backend servisa.

Canvas aplikacija ne sme samostalno određivati finalni broj.

Poznati postojeći flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Acceptance criteria:

- delovodni broj se dodeljuje samo kroz kontrolisani backend proces
- Canvas App šalje zahtev, ali ne generiše finalni broj
- flow vraća finalni broj aplikaciji
- flow vraća standardizovan JSON response
- u slučaju greške korisnik dobija jasnu poruku
- broj se ne može duplirati ni pri istovremenom radu više korisnika

---

## E1-002 — Unique key u listi `RezervisaniBrojevi`

Prioritet:

```text
P0
```

Opis:

Lista `RezervisaniBrojevi` mora imati tehnički ključ koji garantuje jedinstvenost rezervacije broja.

Predloženo polje:

```text
UniqueReservationKey
```

Primer vrednosti:

```text
2026|OS|123
```

Acceptance criteria:

- `UniqueReservationKey` je tekstualno polje
- uključeno je `Enforce unique values`
- polje je indeksirano
- flow koristi ovo polje pri rezervaciji broja
- ako SharePoint odbije duplikat, flow pokreće retry logiku
- svaka neuspešna rezervacija se loguje

---

## E1-003 — Retry logika za race condition

Prioritet:

```text
P0
```

Opis:

Ako dva korisnika istovremeno pokušaju da rezervišu isti broj, sistem mora automatski pokušati sledeći broj.

Minimalna logika:

```text
Try 1 → broj N
Ako duplikat → Try 2 → broj N+1
Ako duplikat → Try 3 → broj N+2
Ako i dalje neuspešno → kontrolisana greška
```

Acceptance criteria:

- retry je ograničen na definisan broj pokušaja
- svaki pokušaj se loguje
- korisnik ne vidi tehničke greške
- flow vraća jasan status:
  - `Success`
  - `DuplicateRetrySuccess`
  - `FailedAfterRetry`
- nema duplih delovodnih brojeva

---

## E1-004 — Unique constraint na finalnom `DelovodniBroj`

Prioritet:

```text
P0
```

Opis:

Finalni poslovni broj mora biti dodatno zaštićen na mestu gde se dokument čuva.

Relevantna biblioteka:

```text
Shared Documents
```

Relevantno polje:

```text
DelovodniBroj
```

Acceptance criteria:

- `DelovodniBroj` je indeksiran
- proverena je mogućnost uključivanja unique constraint-a
- ako unique constraint nije moguć zbog postojećih podataka, dokumentovan je razlog
- postoji alternativna kontrola kroz posebnu registry listu
- svaki finalni upis se proverava pre potvrde korisniku

---

## E1-005 — Audit trag za svaki delovodni broj

Prioritet:

```text
P1
```

Opis:

Svaka rezervacija, dodela i greška vezana za delovodni broj mora biti upisana u audit log.

Predložena lista:

```text
DocCentralAuditLog
```

Minimalna polja:

| Polje | Tip |
|---|---|
| Title | Text |
| CorrelationId | Text |
| EventType | Choice |
| DelovodniBroj | Text |
| ReservedNumber | Number |
| RequestedByEmail | Text |
| FlowName | Text |
| FlowRunId | Text |
| Status | Choice |
| ErrorMessage | Multiple lines of text |
| Created | DateTime |

Acceptance criteria:

- svaki pokušaj rezervacije se loguje
- svaki uspešan broj se loguje
- svaka greška se loguje
- moguće je povezati zapis sa flow run-om
- moguće je povezati zapis sa korisnikom

---

# Epic 2: Document Upload Orchestration

## E2-001 — Standardizovati `Upload Doc` flow

Prioritet:

```text
P1
```

Opis:

Flow `Upload Doc` mora biti dokumentovan i standardizovan.

Poznato:

```text
Upload dokumenta ide preko Power Automate flow-a Upload Doc.
```

Acceptance criteria:

- definisan je ulazni JSON schema
- definisan je izlazni JSON schema
- flow vraća ID dokumenta
- flow vraća URL dokumenta
- flow vraća status
- flow vraća correlation ID
- greške su standardizovane
- upload se povezuje sa delovodnim brojem

---

## E2-002 — Orkestracija upload-a i dodele broja

Prioritet:

```text
P0
```

Opis:

Upload dokumenta i dodela delovodnog broja moraju biti deo kontrolisanog procesa.

Rizik:

```text
Ako upload uspe, a dodela broja ne uspe, može nastati dokument bez validnog delovodnog broja.
Ako dodela broja uspe, a upload ne uspe, može nastati rezervisan broj bez dokumenta.
```

Acceptance criteria:

- definisan je redosled operacija
- definisan je rollback ili status `Failed`
- rezervisani broj ima status
- dokument ima status obrade
- korisnik dobija samo finalni rezultat
- neuspešni pokušaji se mogu naknadno analizirati

---

## E2-003 — Status obrade dokumenta

Prioritet:

```text
P1
```

Predloženo polje:

```text
ProcessingStatus
```

Moguće vrednosti:

- Draft
- Uploading
- NumberReserved
- Completed
- Failed
- Cancelled

Acceptance criteria:

- status postoji u centralnoj registry listi ili biblioteci
- status se ažurira kroz flow
- korisnik vidi razumljivu poruku
- admin može da pronađe neuspele procese

---

# Epic 3: Email Intake Processing

## E3-001 — Standardizovati email intake proces

Prioritet:

```text
P1
```

Opis:

Postoji posebna funkcionalnost gde Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.

Relevantna biblioteka:

```text
EmailDocuments
```

Relevantna polja:

- `Posiljalac`
- `PosiljalacEmail`
- `FileLeafRef`
- `Title`

Acceptance criteria:

- dokumentovan naziv flow-a
- dokumentovana shared mailbox adresa
- čuva se Message ID
- čuva se Attachment ID ili hash
- sprečava se dupla obrada istog emaila
- greške idu u log
- attachment-i bez validnog sadržaja se kontrolišu
- koristi se ista numbering logika kao ručno zavođenje

---

## E3-002 — Dodati zaštitu od duplog email processing-a

Prioritet:

```text
P1
```

Predložena polja:

| Polje | Tip |
|---|---|
| EmailMessageId | Text |
| EmailInternetMessageId | Text |
| AttachmentName | Text |
| AttachmentHash | Text |
| ProcessedAt | DateTime |
| ProcessingStatus | Choice |

Acceptance criteria:

- isti email se ne obrađuje dva puta
- isti attachment se ne zavodi dva puta
- postoji status za greške
- moguće je ponoviti obradu kontrolisano

---

# Epic 4: AppConfig Governance

## E4-001 — JSON schema za `AppConfig`

Prioritet:

```text
P1
```

Opis:

Lista `AppConfig` sadrži šifarnike i konfiguracione JSON zapise.

Relevantna polja:

- `Title`
- `Config`
- `ColumnHeader`

Acceptance criteria:

- definisana je JSON schema
- svaki tip konfiguracije ima poznatu strukturu
- nevalidan JSON se ne prihvata
- postoji validacioni flow ili admin funkcija
- dokumentovan je primer validne konfiguracije

---

## E4-002 — Verzionisanje konfiguracije

Prioritet:

```text
P2
```

Predložena polja:

| Polje | Tip |
|---|---|
| ConfigKey | Text |
| ConfigVersion | Text |
| IsActive | Yes/No |
| Environment | Choice |
| LastExportedAt | DateTime |

Acceptance criteria:

- zna se koja konfiguracija je aktivna
- moguće je vratiti staru verziju
- export uključuje verziju
- konfiguracija se može porediti kroz Git

---

## E4-003 — Export konfiguracije u Git-friendly format

Prioritet:

```text
P2
```

Opis:

Export šifarnika i konfiguracije iz `AppConfig` treba da proizvodi fajlove koji se mogu čuvati u GitHub repozitorijumu.

Predlog strukture:

```text
config/
├── appconfig.json
├── column-headers.json
└── dictionaries/
    ├── document-types.json
    ├── statuses.json
    └── translations.json
```

Acceptance criteria:

- export je deterministički
- JSON je lepo formatiran
- nema tajnih vrednosti
- fajlovi se mogu commit-ovati u Git
- postoji datum i verzija exporta

---

# Epic 5: Security & Permissions

## E5-001 — Read-only SharePoint model za korisnike

Prioritet:

```text
P1
```

Opis:

Korisnici ne treba direktno da menjaju SharePoint liste i biblioteke ako poslovna logika mora ići preko Power Automate flow-ova.

Ciljni model:

```text
Standard users: Read
Service account: Edit / Contribute
Admins: Full Control
```

Acceptance criteria:

- korisnici ne mogu direktno menjati kritične liste
- upisi idu preko flow-ova
- service account je vlasnik ključnih flow-ova
- dokumentovan je permission model
- dokumentovane su SharePoint grupe

---

## E5-002 — Zaštita `RezervisaniBrojevi`

Prioritet:

```text
P0
```

Opis:

Lista `RezervisaniBrojevi` ne sme biti ručno menjana od strane standardnih korisnika.

Acceptance criteria:

- samo service account i admin imaju edit prava
- standardni korisnici nemaju direktan edit
- ručne izmene se loguju
- postoji audit za promene

---

## E5-003 — Zaštita `AppConfig`

Prioritet:

```text
P1
```

Opis:

Greška u `AppConfig` listi može poremetiti rad aplikacije.

Acceptance criteria:

- samo admini mogu menjati konfiguraciju
- izmene se loguju
- pre izmene se validira JSON
- postoji backup/export pre izmene
- postoji rollback procedura

---

# Epic 6: Audit & Monitoring

## E6-001 — Centralni audit log

Prioritet:

```text
P1
```

Predložena lista:

```text
DocCentralAuditLog
```

Acceptance criteria:

- flow-ovi pišu log
- Canvas App dobija correlation ID
- greške su pretražive
- admin može pratiti neuspešne procese
- broj, korisnik i dokument su povezani

---

## E6-002 — Flow run monitoring

Prioritet:

```text
P2
```

Opis:

Potrebno je pratiti neuspešne Power Automate run-ove.

Acceptance criteria:

- postoji dnevni ili periodični pregled neuspešnih run-ova
- kritični flow-ovi imaju alert
- greške vezane za numbering imaju najviši prioritet
- postoji vlasnik procesa

---

## E6-003 — Error response standard

Prioritet:

```text
P1
```

Svi flow-ovi koje poziva Canvas App treba da vraćaju isti format odgovora.

Predlog:

```json
{
  "success": true,
  "status": "Completed",
  "correlationId": "GUID",
  "message": "Document registered successfully.",
  "data": {
    "delovodniBroj": "OS-2026-000123",
    "documentId": 123,
    "documentUrl": "https://..."
  },
  "error": null
}
```

Za grešku:

```json
{
  "success": false,
  "status": "Failed",
  "correlationId": "GUID",
  "message": "Document registration failed.",
  "data": null,
  "error": {
    "code": "NUMBER_RESERVATION_FAILED",
    "details": "Failed after retry."
  }
}
```

Acceptance criteria:

- svi flow-ovi vraćaju isti format
- Canvas App koristi isti parser
- greške su korisnički razumljive
- tehnički detalji se čuvaju u logu

---

# Epic 7: Canvas App Modernization

## E7-001 — Dokumentovati sve ekrane i flow pozive

Prioritet:

```text
P1
```

Poznato:

- aplikacija je Canvas App
- početni ekran je `scrHome`
- postoji ekran/funkcija za zavođenje dokumenta
- pregled dokumenata je direktno u SharePoint listi
- email dokumenti su posebna funkcionalnost preko Power Automate
- export se odnosi na šifarnike i konfiguraciju iz `AppConfig`

Acceptance criteria:

- dokumentovani svi ekrani
- dokumentovani flow pozivi
- dokumentovane kolekcije
- dokumentovane globalne varijable
- dokumentovani error scenariji
- dokumentovana navigacija

---

## E7-002 — Canvas App kao UI, ne kao business transaction engine

Prioritet:

```text
P0
```

Opis:

Canvas App treba da bude korisnički interfejs, dok kritične transakcije treba da se izvršavaju u backend flow-ovima.

Acceptance criteria:

- Canvas App ne generiše finalni broj
- Canvas App ne upisuje direktno kritične podatke
- Canvas App poziva flow
- flow vraća rezultat
- Canvas App prikazuje rezultat

---

## E7-003 — Responsive enterprise UI

Prioritet:

```text
P2
```

Opis:

Nova verzija treba da bude responzivna i stabilna za različite rezolucije.

Acceptance criteria:

- koristi se auto-layout
- nema hardkodovanih širina gde nije potrebno
- forme su čitljive
- greške su jasno prikazane
- korisnik vidi status procesa

---

# Epic 8: SharePoint Data Model Hardening

## E8-001 — Indeksiranje ključnih polja

Prioritet:

```text
P1
```

Kandidati za indeksiranje:

- `DelovodniBroj`
- `EdokumentID`
- `Edokument`
- `Attachment`
- `PosiljalacEmail`
- `RezervisaniBroj`
- `DatumRezervacije`
- `UniqueReservationKey`

Acceptance criteria:

- ključna filter polja su indeksirana
- dokumentovana su indeksirana polja
- provereni su view threshold rizici
- Power Automate filter query koristi indeksirana polja gde je moguće

---

## E8-002 — Registry lista za dokumente

Prioritet:

```text
P1
```

Opis:

Ako dokument biblioteka nije dovoljna kao jedini registry, preporučuje se posebna lista za centralnu evidenciju dokumenata.

Predložena lista:

```text
DocCentralRegistry
```

Minimalna polja:

| Polje | Tip |
|---|---|
| Title | Text |
| DelovodniBroj | Text |
| DocumentLibrary | Text |
| DocumentItemId | Number |
| DocumentUrl | Hyperlink |
| Source | Choice |
| Status | Choice |
| RegisteredByEmail | Text |
| RegisteredAt | DateTime |
| CorrelationId | Text |

Acceptance criteria:

- svako zavođenje ima registry zapis
- registry zapis se povezuje sa dokumentom
- broj je unique
- status je poznat
- audit je moguć

---

# Epic 9: ALM & Deployment

## E9-001 — Solution ALM

Prioritet:

```text
P2
```

Acceptance criteria:

- definisan Dev/Test/Prod proces
- koristi se managed solution za produkciju
- connection references su dokumentovane
- environment variables su dokumentovane
- postoji deployment checklist

---

## E9-002 — GitHub dokumentaciona struktura

Prioritet:

```text
P2
```

Opis:

Dokumentacija se vodi kroz GitHub repozitorijum.

Predložene oblasti:

```text
docs/
architecture/
data-model/
backlog/
prompts/
config/
```

Acceptance criteria:

- svaki dokument je poseban `.md` fajl
- struktura je razumljiva
- dokumentacija se može verzionisati
- Cloud Code može koristiti repozitorijum kao razvojnu osnovu

---

# Epic 10: Reporting & Export

## E10-001 — Export šifarnika i konfiguracije

Prioritet:

```text
P2
```

Opis:

Postojeća funkcionalnost exporta treba da bude standardizovana.

Relevantno:

- `AppConfig`
- `Exports`

Acceptance criteria:

- export ide u biblioteku `Exports`
- export ima verziju
- export ima tip
- export ima datum
- export ima korisnika
- export može da se koristi za Git backup

---

## E10-002 — Administrativni izveštaji

Prioritet:

```text
P3
```

Primeri izveštaja:

- broj zavedenih dokumenata po danu
- broj grešaka po flow-u
- broj email dokumenata
- neuspele rezervacije broja
- najčešće greške
- dokumenti bez finalnog broja

Acceptance criteria:

- izveštaji koriste audit/registry podatke
- admin može da vidi problematične zapise
- moguće je filtrirati po datumu i statusu

---

## 5. Minimalni MVP za novu enterprise verziju

MVP ne treba da obuhvati sve funkcionalnosti, ali mora rešiti kritične rizike.

Minimalni scope:

1. concurrency-safe dodela delovodnog broja
2. `RezervisaniBrojevi` unique key
3. standardizovan `Upload Doc`
4. audit log
5. registry zapis
6. read-only SharePoint model za korisnike
7. osnovna Canvas App forma za zavođenje
8. standardizovan JSON response iz flow-ova

---

## 6. Redosled implementacije

Preporučeni redosled:

1. definisati data model za numbering
2. dodati unique key i indeksiranje
3. refaktorisati numbering flow
4. refaktorisati upload orchestration
5. uvesti audit log
6. uvesti standardni response format
7. prilagoditi Canvas App
8. dokumentovati security model
9. stabilizovati email intake
10. dodati export i monitoring

---

## 7. Zaključak

Enterprise backlog za DocCentral treba da bude vođen jednim osnovnim principom:

```text
Kritične poslovne transakcije moraju biti server-side, auditovane i concurrency-safe.
```

Najvažniji razvojni prioritet je sigurno generisanje i dodela delovodnog broja.

Sve ostale funkcionalnosti treba graditi oko tog principa.
