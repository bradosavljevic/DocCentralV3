# 05 — Document libraries analiza

## 1. Svrha dokumenta

Ovaj dokument opisuje identifikovane SharePoint document libraries u okviru rešenja **DocCentral v6.0**.

Obuhvaćene biblioteke:

- `Shared Documents`
- `EmailDocuments`
- `Exports`

Cilj dokumenta je da definiše:

- potvrđene biblioteke i njihova polja
- pretpostavljenu ulogu svake biblioteke
- rizike trenutnog modela
- preporuke za enterprise verziju
- pravila za dokumentni lifecycle
- otvorena pitanja za dalju analizu

---

## 2. Status analize

Status:

```text
DELIMIČNO POTVRĐENO
```

Razlog:

- SharePoint REST metadata potvrđuje postojanje više document libraries.
- Metadata potvrđuje nazive i tipove pojedinih kolona.
- Nisu dostavljeni stvarni itemi/fajlovi iz biblioteka.
- Nisu dostavljeni Power Automate flow-ovi koji koriste biblioteke.
- Nije dostavljena Canvas app logika koja radi sa dokumentima.
- Zato je deo zaključaka potvrđen, a deo je arhitektonska pretpostavka.

---

## 3. Identifikovane document libraries

| Biblioteka | Scope | Status |
|---|---|---|
| `Shared Documents` | `/sites/DocumentCentralv6.0/Shared Documents` | Potvrđeno |
| `EmailDocuments` | `/sites/DocumentCentralv6.0/EmailDocuments` | Potvrđeno |
| `Exports` | `/sites/DocumentCentralv6.0/Exports` | Potvrđeno |

REST base:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 4. Visok nivo uloge biblioteka

Pretpostavljeni model:

```text
EmailDocuments
      |
      | email intake / dokumenti primljeni emailom
      v
Power Automate / Power Apps obrada
      |
      v
Shared Documents
      |
      | finalno zavedeni dokumenti
      v
Exports
      |
      | exporti / izveštaji / generisani fajlovi
      v
Korisnici / audit / arhiva
```

Status: **pretpostavka**

Razlog: nazivi biblioteka i polja ukazuju na ovakav model, ali tokovi nisu dostavljeni.

---

# 5. Shared Documents

## 5.1 Svrha

`Shared Documents` je najverovatnije glavna document library u kojoj se čuvaju dokumenti koji su deo DocCentral procesa.

Status: **delimično potvrđeno**

Razlog:

- Biblioteka postoji.
- Ima polja vezana za dokument i delovodni broj.
- Ne postoji dostavljen dokaz da su svi finalni dokumenti baš u ovoj biblioteci, ali polje `DelovodniBroj` snažno ukazuje na tu ulogu.

---

## 5.2 Potvrđeni scope

```text
/sites/DocumentCentralv6.0/Shared Documents
```

---

## 5.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Status |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Potvrđeno |
| `Title` | Title | Single line of text | Ne | Ne | Potvrđeno |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Potvrđeno |
| `DelovodniBroj` | DelovodniBroj | Single line of text | Ne | Ne | Potvrđeno |
| `EdokumentID` | EdokumentID | Single line of text | Ne | Ne | Potvrđeno |
| `Edokument` | Edokument | Yes/No | Ne | Ne | Potvrđeno |
| `Attachment` | Attachment | Yes/No | Ne | Ne | Potvrđeno |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Potvrđeno |
| `ContentType` | Content Type | Computed | Ne | Ne | Potvrđeno |

---

## 5.4 Ključna potvrđena polja

### 5.4.1 `FileLeafRef`

```text
InternalName: FileLeafRef
DisplayName: Name
TypeAsString: File
Required: true
```

Namena:

- naziv fajla
- osnovni file identity u biblioteci
- koristi se pri upload-u i rename-u fajla

---

### 5.4.2 `DelovodniBroj`

```text
InternalName: DelovodniBroj
DisplayName: DelovodniBroj
TypeAsString: Text
MaxLength: 255
```

Namena:

- čuva delovodni broj povezan sa dokumentom
- kritično polje za pisarnicu
- mora biti jedinstveno u enterprise verziji

Status: **potvrđeno da polje postoji**

Napomena:

Iz metadata nije potvrđeno da je `EnforceUniqueValues = true`. Zato se ne može tvrditi da SharePoint trenutno sprečava duplikate.

---

### 5.4.3 `Edokument`

```text
InternalName: Edokument
DisplayName: Edokument
TypeAsString: Boolean
DefaultValue: 0
```

Namena:

- označava da li je dokument elektronski dokument
- default vrednost je `false`

Status: **potvrđeno**

---

### 5.4.4 `EdokumentID`

```text
InternalName: EdokumentID
DisplayName: EdokumentID
TypeAsString: Text
MaxLength: 255
```

Pretpostavljena namena:

- identifikator elektronskog dokumenta
- moguća veza ka eksternom sistemu ili elektronskom dokumentnom zapisu

Status: **pretpostavka**

---

### 5.4.5 `Attachment`

```text
InternalName: Attachment
DisplayName: Attachment
TypeAsString: Boolean
DefaultValue: 0
```

Pretpostavljena namena:

- označava da je fajl prilog, a ne glavni dokument
- može se koristiti za razlikovanje osnovnog dokumenta i dodatnih fajlova

Status: **pretpostavka**

---

## 5.5 Rizici za `Shared Documents`

### 5.5.1 `DelovodniBroj` nije potvrđeno unikatan

Najveći rizik:

```text
Dva dokumenta mogu dobiti isti DelovodniBroj ako ne postoji unique constraint i concurrency-safe generisanje.
```

Obavezno za novu verziju:

- `DelovodniBroj` mora biti jedinstven
- generisanje mora biti server-side
- mora postojati retry logika
- mora postojati audit log
- Canvas app ne sme garantovati jedinstvenost sama

---

### 5.5.2 Metadata i dokument mogu biti razdvojeni

Mogući rizik:

- dokument je uploadovan, ali metadata nije kompletan
- metadata je kreiran, ali fajl nije povezan
- dokument dobije delovodni broj, ali upload ne uspe
- upload uspe, ali flow ne ažurira status

Potrebno je definisati transakcioni obrazac.

---

### 5.5.3 SharePoint nije transakciona baza

SharePoint Online nema klasičnu SQL transakciju preko više lista/biblioteka.

Zato treba dizajnirati kompenzacionu logiku:

- ako upload ne uspe, rollback ili status `Greška`
- ako numbering uspe, a dokument ne uspe, rezervacija mora ostati auditovana
- ako finalni update ne uspe, retry
- ako retry ne uspe, incident log

---

## 5.6 Preporuke za `Shared Documents`

| Oblast | Preporuka |
|---|---|
| Jedinstvenost | Uključiti unique constraint na `DelovodniBroj` ako je tehnički moguće |
| Indexing | Indeksirati `DelovodniBroj`, `Edokument`, `Created`, `Modified` |
| Metadata | Jasno definisati obavezna polja |
| Lifecycle | Uvesti status dokumenta |
| Audit | Svaki upload/update treba auditovati |
| Permissions | Korisnici ne treba direktno da menjaju kritična metadata polja |
| Flow ownership | Upis treba da radi servisni nalog kroz Power Automate |
| Naming | Definisati pravila imenovanja fajlova |
| Retention | Razmotriti retention labels |
| Error handling | Dodati kontrolisane greške i retry logiku |

---

# 6. EmailDocuments

## 6.1 Svrha

`EmailDocuments` je najverovatnije biblioteka za dokumente koji ulaze u sistem preko email procesa.

Status: **pretpostavka**

Razlog:

- biblioteka se zove `EmailDocuments`
- ima polja `PosiljalacEmail` i `Posiljalac`
- to ukazuje na email intake scenario

---

## 6.2 Potvrđeni scope

```text
/sites/DocumentCentralv6.0/EmailDocuments
```

---

## 6.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Status |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Potvrđeno |
| `Title` | Title | Single line of text | Ne | Ne | Potvrđeno |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Potvrđeno |
| `PosiljalacEmail` | PosiljalacEmail | Single line of text | Ne | Ne | Potvrđeno |
| `Posiljalac` | Posiljalac | Single line of text | Ne | Ne | Potvrđeno |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Potvrđeno |
| `ContentType` | Content Type | Computed | Ne | Ne | Potvrđeno |

---

## 6.4 Ključna potvrđena polja

### 6.4.1 `PosiljalacEmail`

```text
InternalName: PosiljalacEmail
DisplayName: PosiljalacEmail
TypeAsString: Text
MaxLength: 255
```

Namena:

- čuva email adresu pošiljaoca
- koristi se za identifikaciju izvora dokumenta
- može se koristiti za pravila automatske klasifikacije

Status: **potvrđeno da polje postoji**

---

### 6.4.2 `Posiljalac`

```text
InternalName: Posiljalac
DisplayName: Posiljalac
TypeAsString: Text
MaxLength: 255
```

Namena:

- čuva ime ili naziv pošiljaoca
- može se koristiti u prikazu korisniku ili u email obradi

Status: **potvrđeno da polje postoji**

---

## 6.5 Pretpostavljeni email intake proces

Mogući proces:

```text
Email stigne u mailbox
      |
      v
Power Automate flow čita email
      |
      v
Prilozi se snimaju u EmailDocuments
      |
      v
Upisuju se PosiljalacEmail i Posiljalac
      |
      v
Korisnik pregleda dokumente u Canvas app
      |
      v
Korisnik ili flow zavodi dokument
      |
      v
Dokument se prebacuje ili kopira u Shared Documents
```

Status: **pretpostavka**

---

## 6.6 Rizici za `EmailDocuments`

### 6.6.1 Nevalidni ili nepotpuni email podaci

Mogući problemi:

- email nema pošiljaoca
- pošiljalac je shared mailbox
- attachment nema očekivanu ekstenziju
- fajl je prevelik
- fajl je duplikat
- attachment je inline image, a ne stvarni poslovni dokument

---

### 6.6.2 Bezbednosni rizik email priloga

Email attachments mogu sadržati:

- malware
- pogrešan dokument
- poverljive podatke
- nevalidan format
- duplikate

Preporuke:

- dozvoliti samo podržane tipove fajlova
- dodati proveru veličine fajla
- dodati skeniranje/bezbednosnu proveru gde je moguće
- auditovati izvor emaila
- označiti dokumente kao `Unprocessed`, `Processed`, `Rejected`

---

### 6.6.3 Duplikati

Mogući izvori duplikata:

- isti email obrađen više puta
- isti attachment poslat više puta
- retry flow-a kreira duplikat
- korisnik ručno uploaduje isti dokument

Preporuke:

- čuvati email message id
- čuvati attachment id
- čuvati hash fajla ako je moguće
- uvesti status obrade
- uvesti jedinstveni intake key

---

## 6.7 Preporuke za `EmailDocuments`

Predložena dodatna polja:

| Polje | Tip | Namena |
|---|---|---|
| `EmailMessageId` | Single line of text | Jedinstveni ID email poruke |
| `EmailConversationId` | Single line of text | Thread/conversation ID |
| `AttachmentId` | Single line of text | ID priloga |
| `AttachmentFileNameOriginal` | Single line of text | Originalni naziv priloga |
| `ReceivedDateTime` | Date and Time | Kada je email primljen |
| `ProcessingStatus` | Choice | New/Processing/Processed/Rejected/Error |
| `ProcessingError` | Multiple lines of text | Greška obrade |
| `ProcessedOn` | Date and Time | Kada je obrađeno |
| `ProcessedByFlowRunId` | Single line of text | Flow run ID |
| `SourceMailbox` | Single line of text | Mailbox iz kog je preuzeto |
| `FileHash` | Single line of text | Hash za detekciju duplikata |

---

# 7. Exports

## 7.1 Svrha

`Exports` je najverovatnije biblioteka za fajlove koje sistem generiše kao export, izveštaj ili paket podataka.

Status: **pretpostavka**

Razlog:

- naziv biblioteke je `Exports`
- potvrđena su osnovna document library polja
- nisu dostavljeni flow-ovi koji kreiraju export

---

## 7.2 Potvrđeni scope

```text
/sites/DocumentCentralv6.0/Exports
```

---

## 7.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Status |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Potvrđeno |
| `Title` | Title | Single line of text | Ne | Ne | Potvrđeno |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Potvrđeno |
| `ContentType` | Content Type | Computed | Ne | Ne | Potvrđeno |

---

## 7.4 Pretpostavljeni export proces

Mogući proces:

```text
Korisnik pokrene export iz Canvas app
      |
      v
Power Automate flow prikupi podatke
      |
      v
Generiše Excel/PDF/CSV/ZIP
      |
      v
Snima fajl u Exports biblioteku
      |
      v
Vraća link korisniku
      |
      v
Korisnik preuzima fajl
```

Status: **pretpostavka**

---

## 7.5 Rizici za `Exports`

Mogući rizici:

- export fajlovi ostaju trajno bez retention pravila
- fajlovi mogu sadržati poverljive podatke
- korisnici mogu videti exporte drugih korisnika
- nema statusa generisanja
- nema evidencije ko je pokrenuo export
- nema automatskog brisanja starih fajlova
- veliki fajlovi mogu uticati na storage

---

## 7.6 Preporuke za `Exports`

Predložena dodatna polja:

| Polje | Tip | Namena |
|---|---|---|
| `ExportType` | Choice | Tip exporta |
| `RequestedByEmail` | Single line of text | Ko je pokrenuo export |
| `RequestedOn` | Date and Time | Kada je pokrenuto |
| `GeneratedOn` | Date and Time | Kada je generisano |
| `ExportStatus` | Choice | Requested/Processing/Completed/Failed |
| `FlowRunId` | Single line of text | Flow run koji je generisao fajl |
| `SourceFilterJson` | Multiple lines of text | Filteri korišćeni za export |
| `ExpiresOn` | Date and Time | Kada export ističe |
| `DownloadCount` | Number | Broj preuzimanja |
| `ErrorMessage` | Multiple lines of text | Greška ako nije uspelo |

---

# 8. Cross-library lifecycle

## 8.1 Predloženi dokumentni lifecycle

Za enterprise verziju preporučuje se jasan lifecycle:

```text
1. Intake
   - dokument dolazi iz emaila, ručnog upload-a ili integracije

2. Validation
   - proverava se format, obavezna metadata i poslovna pravila

3. Number reservation
   - backend rezerviše delovodni broj

4. Registration
   - dokument se zavodi i dobija finalni DelovodniBroj

5. Storage
   - dokument se čuva u finalnoj biblioteci

6. Processing
   - odobrenje, klasifikacija, arhiviranje ili slanje dalje

7. Export / reporting
   - po potrebi se generišu exporti

8. Archive / retention
   - dokument ulazi u retention/arhivski režim
```

---

## 8.2 Predloženi statusi dokumenta

Preporučeni statusi:

| Status | Opis |
|---|---|
| `Draft` | Dokument je započet ali nije zaveden |
| `PendingValidation` | Čeka validaciju |
| `ValidationFailed` | Validacija nije uspela |
| `NumberReserved` | Delovodni broj je rezervisan |
| `Registered` | Dokument je zaveden |
| `Processing` | Dokument je u obradi |
| `Approved` | Dokument je odobren |
| `Rejected` | Dokument je odbijen |
| `Archived` | Dokument je arhiviran |
| `Error` | Došlo je do tehničke greške |

---

## 8.3 Kritično pravilo za delovodni broj

Ovo je enterprise zahtev najvišeg prioriteta:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

Za document libraries to znači:

- `DelovodniBroj` u finalnoj biblioteci mora biti jedinstven
- broj se ne sme dodeljivati samo u Canvas app
- broj se mora dodeljivati u backend flow/service logici
- mora postojati retry za collision
- mora postojati audit log za svaku rezervaciju
- mora postojati kontrola neuspelih rezervacija
- korisnik mora dobiti jasan odgovor ako zavođenje ne uspe

---

# 9. Preporučena ciljna struktura biblioteka

## 9.1 Minimalna enterprise struktura

```text
DocumentCentral site
│
├── Documents
│   └── finalni registrovani dokumenti
│
├── IntakeDocuments
│   └── dokumenti koji čekaju obradu
│
├── EmailDocuments
│   └── dokumenti dobijeni emailom
│
├── Exports
│   └── generisani export fajlovi
│
├── Archive
│   └── arhivirani dokumenti, ako se koristi posebna biblioteka
│
└── ErrorDocuments
    └── dokumenti koji su pali u tehničku grešku
```

Napomena:

Ovo je preporuka. Trenutno su potvrđene biblioteke `Shared Documents`, `EmailDocuments` i `Exports`.

---

## 9.2 Alternativni model

Umesto više biblioteka, moguće je koristiti jednu glavnu biblioteku sa jasnim statusima i content type-ovima.

Primer:

```text
Documents
├── Content Type: IncomingDocument
├── Content Type: RegisteredDocument
├── Content Type: Attachment
├── Content Type: Export
└── Content Type: ArchivedDocument
```

Prednost:

- centralizovan dokumentni model
- lakše pretrage
- lakše permission upravljanje

Mana:

- može postati kompleksno
- zahteva dobar content type i metadata dizajn
- može imati više rizika kod velikog broja fajlova

---

# 10. Permission model preporuke

Za enterprise verziju:

| Uloga | Preporučena prava |
|---|---|
| Obični korisnik | Read nad bibliotekama, bez direktnog edit-a kritičnih polja |
| Referent / zavođenje | Kroz aplikaciju može pokrenuti zavođenje |
| Power Automate service account | Edit/Contribute ili veća prava za upis |
| Admin | Full Control |
| Auditor | Read nad dokumentima i audit logom |
| Export korisnik | Pristup samo sopstvenim exportima ili prema roli |

Ključno:

```text
Ako korisnici imaju Read Only prava nad SharePoint-om,
sve Create/Edit/Delete aktivnosti moraju ići kroz Power Automate.
```

---

# 11. Naming convention preporuke

## 11.1 Dokumenti

Preporučeni format:

```text
{DelovodniBroj}_{DocumentType}_{YYYYMMDD}_{OriginalFileName}
```

Primer:

```text
OP-2026-000123_Ugovor_20260430_partner.pdf
```

## 11.2 Email intake fajlovi

Preporučeni format:

```text
EMAIL_{ReceivedDateTime}_{SenderDomain}_{OriginalFileName}
```

Primer:

```text
EMAIL_20260430_153012_partner-com_invoice.pdf
```

## 11.3 Export fajlovi

Preporučeni format:

```text
EXPORT_{ExportType}_{RequestedBy}_{YYYYMMDD_HHMMSS}.xlsx
```

Primer:

```text
EXPORT_Delovodnik_bozidar_20260430_170000.xlsx
```

---

# 12. Retention preporuke

Za dokumentnu pisarnicu treba razmotriti:

- retention labels
- records management
- archive policy
- zabranu brisanja registrovanih dokumenata
- audit trail
- legal hold scenarije
- rokove čuvanja po tipu dokumenta

Minimalno pravilo:

```text
Registrovani dokument sa dodeljenim delovodnim brojem ne sme biti fizički obrisan bez kontrolisane procedure.
```

---

# 13. Audit preporuke

Svaka važna operacija treba da se auditira:

| Operacija | Audit |
|---|---|
| Upload dokumenta | Da |
| Promena metadata | Da |
| Rezervacija delovodnog broja | Da |
| Finalno zavođenje | Da |
| Greška pri zavođenju | Da |
| Promena statusa | Da |
| Export | Da |
| Brisanje ili arhiviranje | Da |
| Promena dozvola | Da |

Predložena posebna lista:

```text
AuditLog
```

Predložena polja:

| Polje | Tip |
|---|---|
| `EventType` | Choice |
| `EntityType` | Choice |
| `EntityId` | Single line of text |
| `DocumentId` | Number |
| `DelovodniBroj` | Single line of text |
| `UserEmail` | Single line of text |
| `FlowRunId` | Single line of text |
| `PayloadJson` | Multiple lines of text |
| `ErrorMessage` | Multiple lines of text |
| `CreatedOn` | Date and Time |

---

# 14. Performance preporuke

SharePoint document libraries treba projektovati za rast.

Preporuke:

- indeksirati kolone koje se koriste u filterima
- ne prikazivati sve dokumente bez filtera
- koristiti delegabilne filtere u Power Apps
- izbegavati velike nekontrolisane galerije
- koristiti paging
- odvojiti archive od aktivnih dokumenata ako broj fajlova značajno raste
- planirati view-eve prema najčešćim poslovnim scenarijima
- koristiti flow za kompleksne upite ako Power Apps delegacija nije dovoljna

---

# 15. Integritet podataka

Minimalni integritet mora obuhvatiti:

| Pravilo | Opis |
|---|---|
| `DelovodniBroj` jedinstven | Ne sme biti duplikata |
| Dokument ima status | Svaki dokument mora imati lifecycle status |
| Dokument ima izvor | Email/manual/integration |
| Dokument ima audit | Svaka promena mora biti zapisana |
| Registrovan dokument je zaključan | Kritična metadata se ne menja direktno |
| Greške su vidljive | Neuspešne obrade imaju error zapis |
| Export ima vlasnika | Mora se znati ko ga je kreirao |

---

# 16. Known issues / tehnički dug

Na osnovu metadata analize mogu se izdvojiti sledeći potencijalni tehnički dugovi:

| Oblast | Problem | Status |
|---|---|---|
| Numbering | Nije potvrđen unique constraint na `DelovodniBroj` | Rizik |
| Lifecycle | Nije potvrđeno status polje u bibliotekama | Nepoznato |
| Audit | Nije potvrđena audit lista | Nepoznato |
| Email intake | Nema potvrđen `EmailMessageId` | Rizik |
| Exports | Nema potvrđen retention model | Rizik |
| Permissions | Nije poznat permission model | Nepoznato |
| Metadata | Nije poznato koja polja su obavezna u aplikaciji | Nepoznato |
| Flow logic | Nisu dostavljeni flow-ovi | Nepoznato |

---

# 17. Preporuke za novu verziju

Za novu enterprise verziju preporučuje se:

1. Definisati jedinstven dokumentni lifecycle.
2. Uvesti obavezno status polje.
3. Uvesti centralni audit log.
4. Uvesti concurrency-safe servis za delovodne brojeve.
5. Uključiti unique constraint na finalni `DelovodniBroj`.
6. Razdvojiti intake dokumente od registrovanih dokumenata.
7. Uvesti jasnu email intake deduplikaciju.
8. Uvesti export lifecycle i automatsko čišćenje starih exporta.
9. Ograničiti direktan edit nad kritičnim bibliotekama.
10. Sve upise raditi kroz Power Automate service account.
11. Dokumentovati svaku biblioteku i svaku kolonu u GitHub-u.
12. Uvesti JSON/Markdown dokumentaciju za data model.

---

# 18. Otvorena pitanja

## 18.1 Shared Documents

1. Da li je `Shared Documents` finalna biblioteka registrovanih dokumenata?
2. Da li `DelovodniBroj` ima unique constraint?
3. Da li postoji status dokumenta u biblioteci?
4. Da li postoji veza između dokumenta i predmeta?
5. Da li se prilozi čuvaju u istoj biblioteci?
6. Šta tačno označava polje `Attachment`?
7. Šta tačno označava polje `Edokument`?
8. Šta je izvor vrednosti za `EdokumentID`?
9. Da li se dokumenti rename-uju nakon zavođenja?
10. Da li postoji retention label za registrovane dokumente?

## 18.2 EmailDocuments

1. Koji flow puni `EmailDocuments`?
2. Da li se čuvaju email body i attachment metadata?
3. Da li se čuva email message id?
4. Da li postoji deduplikacija?
5. Da li korisnik ručno bira koje email dokumente zavodi?
6. Da li se dokument kopira ili premešta u `Shared Documents`?
7. Šta se dešava sa obrađenim email dokumentima?
8. Da li postoje rejected/error statusi?

## 18.3 Exports

1. Ko pokreće export?
2. Koji format exporta postoji?
3. Da li export sadrži poverljive podatke?
4. Ko ima pravo da vidi export?
5. Da li se exporti automatski brišu?
6. Da li postoji limit veličine exporta?
7. Da li se export status prikazuje korisniku?

---

# 19. Zaključak

U rešenju DocCentral v6.0 potvrđene su tri važne document libraries:

- `Shared Documents`
- `EmailDocuments`
- `Exports`

`Shared Documents` je najvažnija biblioteka jer sadrži polje `DelovodniBroj`, koje je direktno povezano sa osnovnim poslovnim zahtevom pisarnice.

`EmailDocuments` najverovatnije predstavlja ulazni kanal za dokumente dobijene emailom.

`Exports` najverovatnije predstavlja biblioteku za generisane export fajlove.

Najveći arhitektonski rizik je jedinstvenost delovodnog broja i integritet dokumentnog lifecycle-a.

Za enterprise verziju potrebno je formalizovati:

- lifecycle dokumenta
- statusni model
- audit
- permission model
- deduplikaciju email intake procesa
- export retention
- concurrency-safe numbering
- unique constraint na finalnom delovodnom broju

---

## 20. Sledeći dokument

Sledeći dokument:

```text
06-numbering-and-concurrency.md
```

Taj dokument treba posebno i detaljno da obradi:

- generisanje delovodnog broja
- `RezervisaniBrojevi`
- concurrency-safe model
- optimistic locking
- retry logiku
- unique constraint
- audit
- greške i rollback
