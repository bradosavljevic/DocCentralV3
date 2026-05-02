# Numbering Sequence — Concurrency-safe dodela delovodnog broja

## 1. Svrha dokumenta

Ovaj dokument opisuje preporučenu sekvencu za bezbedno generisanje, rezervaciju i potvrdu delovodnog broja u rešenju **DocCentral**.

Glavni zahtev:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovaj dokument je direktno povezan sa:

```text
docs/06-numbering-and-concurrency.md
architecture/target-architecture.md
data-model/rezervisani-brojevi.md
```

---

## 2. Problem koji se rešava

Kod sistema za zavođenje dokumenata najveći rizik je race condition.

Primer problema:

```text
Korisnik A pročita da je poslednji broj 100.
Korisnik B u isto vreme pročita da je poslednji broj 100.

Korisnik A predloži broj 101.
Korisnik B predloži broj 101.

Ako nema server-side zaštite, oba dokumenta mogu dobiti isti delovodni broj.
```

Ovo ne sme biti dozvoljeno.

---

## 3. Osnovno pravilo

Delovodni broj ne sme biti garantovan na nivou Canvas aplikacije.

Canvas aplikacija sme da:

- pokrene proces
- pošalje podatke
- prikaže rezultat
- prikaže grešku

Canvas aplikacija ne sme da:

- samostalno potvrdi sledeći broj
- garantuje jedinstvenost
- oslanja se na lokalnu kolekciju kao konačan izvor istine
- finalno upiše broj bez backend potvrde

---

## 4. Komponente procesa

### 4.1 Power Apps Canvas App

Uloga:

- korisnik unosi podatke
- korisnik dodaje dokument
- aplikacija poziva Power Automate flow
- aplikacija prikazuje rezultat

Potvrđeno:

```text
Aplikacija je Canvas App.
Početni ekran je scrHome.
Postoji ekran za zavođenje dokumenta.
```

---

### 4.2 Upload Doc flow

Potvrđeno:

```text
Upload dokumenta ide preko Power Automate flow-a Upload Doc.
```

Uloga:

- prima fajl i metadata iz Canvas aplikacije
- pokreće backend proces
- ne sme da dozvoli nekontrolisan direktan upis delovodnog broja iz Canvas aplikacije

---

### 4.3 Numbering flow

Potvrđen naziv flow-a:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Uloga:

- rešava race condition
- dodeljuje delovodni broj
- mora raditi server-side
- mora imati retry / locking logiku
- mora vratiti rezultat nazad pozivaocu

---

### 4.4 SharePoint lista RezervisaniBrojevi

Identifikovana lista:

```text
RezervisaniBrojevi
```

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

```text
Lista je namenjena rezervaciji brojeva ili evidenciji rezervisanih brojeva.
```

Status:

```text
POTVRĐENO: lista i polja postoje.
PRETPOSTAVKA: detaljna logika korišćenja liste nije potvrđena bez flow definicije.
```

---

## 5. Preporučena sekvenca zavođenja dokumenta

### 5.1 Visok nivo

```text
1. Korisnik popunjava formu u Canvas aplikaciji.
2. Canvas aplikacija poziva Upload Doc flow.
3. Upload Doc flow čuva dokument ili priprema dokument za zavođenje.
4. Flow poziva numbering logiku.
5. Numbering flow rezerviše sledeći slobodan broj.
6. Sistem upisuje delovodni broj na dokument / stavku.
7. Sistem potvrđuje rezervaciju kao iskorišćenu.
8. Sistem upisuje audit log.
9. Canvas aplikacija dobija rezultat.
```

---

## 6. Detaljna sekvenca

```text
Canvas App
   |
   | Submit document registration request
   v
Upload Doc Flow
   |
   | Validate payload
   | Validate file
   | Generate CorrelationId
   v
Numbering Flow
   |
   | Read current numbering state
   | Try reserve next number
   | If conflict: retry with next number
   | If success: return reserved number
   v
Upload / Update Document Metadata
   |
   | Set DelovodniBroj
   | Set document metadata
   v
Commit Number Reservation
   |
   | Mark number as used / committed
   v
Audit Log
   |
   | Save operation result
   v
Response to Canvas App
```

---

## 7. Predloženi statusi broja

Za enterprise stabilnost, broj ne treba biti samo običan red u listi.

Treba imati status.

Predloženi statusi:

| Status | Opis |
|---|---|
| `Reserved` | Broj je rezervisan, ali još nije finalno povezan sa dokumentom |
| `Committed` | Broj je iskorišćen i povezan sa dokumentom |
| `Cancelled` | Rezervacija je ručno ili sistemski poništena |
| `Expired` | Rezervacija je istekla jer proces nije završen |
| `Failed` | Proces je neuspešan i zahteva proveru |

Ako postojeća lista `RezervisaniBrojevi` nema status, preporuka je dodati status kolonu ili uvesti novu listu `NumberReservations`.

---

## 8. Preporučena nova lista NumberReservations

Ako postojeća lista `RezervisaniBrojevi` nije dovoljna, preporučuje se nova lista:

```text
NumberReservations
```

Predložena polja:

| Internal name | Tip | Obavezno | Namena |
|---|---|---|---|
| `Title` | Text | Da | Finalni delovodni broj |
| `NumberValue` | Number | Da | Numerički deo broja |
| `NumberPrefix` | Text | Ne | Prefiks broja |
| `NumberYear` | Number | Da | Godina |
| `ReservationStatus` | Choice | Da | Reserved / Committed / Cancelled / Expired / Failed |
| `ReservedByEmail` | Text | Da | Korisnik koji je pokrenuo proces |
| `ReservedAt` | DateTime | Da | Vreme rezervacije |
| `CommittedAt` | DateTime | Ne | Vreme potvrde |
| `DocumentItemId` | Number | Ne | ID dokumenta ili item-a |
| `DocumentLibrary` | Text | Ne | Biblioteka dokumenta |
| `CorrelationId` | Text | Da | Jedinstven ID operacije |
| `RetryCount` | Number | Ne | Broj pokušaja |
| `ErrorMessage` | Multiple lines | Ne | Greška ako postoji |
| `Payload` | Multiple lines | Ne | JSON zahtev / tehnički trag |

Obavezno:

```text
Title mora imati unique constraint.
```

Alternativno:

```text
Kombinacija NumberPrefix + NumberYear + NumberValue mora biti jedinstvena.
```

Pošto SharePoint nema composite unique constraint, praktično rešenje je da se jedinstvena vrednost upiše u `Title` ili posebno polje `UniqueNumberKey`.

Primer:

```text
OS-2026-000123
```

---

## 9. Unique key strategija

Predloženo polje:

```text
UniqueNumberKey
```

Primeri vrednosti:

```text
OS-2026-000001
OS-2026-000002
OS-2026-000003
```

ili ako se koristi pravni / poslovni format:

```text
Os.Del.Br.-2026-000001
```

Pravilo:

```text
UniqueNumberKey mora biti obavezan i jedinstven.
```

Prednost:

- lako se proverava duplikat
- SharePoint može imati unique constraint nad jednim tekstualnim poljem
- backend flow može jednostavno da uhvati konflikt
- retry logika je jasna

---

## 10. Retry logika

### 10.1 Osnovni princip

Ako pokušaj rezervacije broja ne uspe jer broj već postoji, flow ne sme odmah završiti greškom.

Treba da:

```text
1. pročita / izračuna sledeći kandidat broj
2. pokuša upis sa unique key
3. ako upis ne uspe zbog duplikata, poveća broj
4. pokuša ponovo
5. nakon maksimalnog broja pokušaja vrati kontrolisanu grešku
```

---

### 10.2 Predloženi retry parametri

| Parametar | Predlog |
|---|---|
| Maksimalan broj pokušaja | 3 do 5 |
| Delay između pokušaja | 200ms do 1000ms |
| Error code | `NUMBER_RESERVATION_CONFLICT` |
| Final error code | `NUMBER_RESERVATION_FAILED` |
| Obavezan audit | Da |

---

### 10.3 Pseudologika

```text
Set retryCount = 0
Set maxRetries = 5

Do until reservationSuccess = true OR retryCount >= maxRetries:

    Get next candidate number

    Try create reservation item with UniqueNumberKey

    If create succeeds:
        reservationSuccess = true
        return reserved number

    If create fails because duplicate:
        retryCount = retryCount + 1
        candidateNumber = candidateNumber + 1
        continue

    If create fails because other error:
        log error
        return controlled failure
```

---

## 11. Optimistic locking model

Ako se koristi centralni counter item, potrebno je koristiti optimistic concurrency.

Primer modela:

```text
Lista: NumberCounters

Polja:
- Title
- CurrentNumber
- Year
- Prefix
- Modified
- ETag
```

Sekvenca:

```text
1. Flow pročita counter item i ETag.
2. Flow izračuna sledeći broj.
3. Flow pokušava update counter item-a sa If-Match = prethodni ETag.
4. Ako se ETag promenio, neko je već izmenio counter.
5. Flow ponavlja čitanje i pokušava ponovo.
```

Prednost:

- postoji jedan centralni counter
- nema oslanjanja na Max() iz liste
- bolje za visoku konkurentnost

Rizik:

- potrebno je pažljivo rukovati SharePoint HTTP akcijama
- potrebno je jasno obraditi 412 Precondition Failed
- potrebno je auditovati svaki retry

---

## 12. Create-first reservation model

Alternativni i praktičan model u SharePoint-u:

```text
Umesto update centralnog counter-a, flow pokušava da kreira rezervaciju sa unique key.
```

Primer:

```text
UniqueNumberKey = OS-2026-000101
```

Ako SharePoint odbije kreiranje zbog duplikata, flow proba:

```text
OS-2026-000102
```

Prednost:

- koristi se SharePoint unique constraint
- jednostavnije za audit
- svaka rezervacija je poseban zapis
- lakše je videti istoriju

Rizik:

- potrebno je imati dobru logiku za izbor sledećeg kandidata
- potrebno je čistiti expired / failed rezervacije
- mora se sprečiti nekontrolisano preskakanje brojeva ako je to poslovno važno

---

## 13. Preporučeni pristup za DocCentral

Za DocCentral se preporučuje kombinovani pristup:

```text
1. Numbering flow kao jedini servis za dodelu broja.
2. SharePoint unique constraint nad finalnim ključem.
3. Rezervacija broja pre finalnog upisa.
4. Commit status nakon uspešnog upisa dokumenta.
5. Retry ako dođe do konflikta.
6. Audit log za svaki pokušaj.
```

Minimalni obavezni uslovi:

```text
- Canvas App ne generiše finalni broj.
- Power Automate generiše i rezerviše broj.
- DelovodniBroj ili UniqueNumberKey je jedinstven u SharePoint-u.
- Postoji log neuspešnih i uspešnih pokušaja.
- Postoji kontrolisan response u Canvas aplikaciju.
```

---

## 14. Predložena sekvenca sa statusima

```text
1. Start
   Status: New request

2. Validate input
   Status: Validated

3. Reserve number
   Status: Reserved

4. Upload / update document
   Status: DocumentCreated

5. Commit number
   Status: Committed

6. Write audit
   Status: Logged

7. Return success
   Status: Completed
```

Ako dođe do greške:

```text
1. Error detected
2. Write ErrorLog
3. Update reservation status:
   - Failed
   - Cancelled
   - Expired
4. Return controlled error
```

---

## 15. Greške koje treba predvideti

| Greška | Primer | Reakcija |
|---|---|---|
| Duplicate number | Broj već postoji | Retry |
| SharePoint throttling | 429 / 503 | Retry sa delay |
| ETag conflict | 412 | Ponovno čitanje i retry |
| Upload failed | Dokument nije sačuvan | Obeležiti rezervaciju kao Failed |
| Metadata update failed | Dokument postoji, broj nije upisan | Log + korektivna akcija |
| Flow timeout | Canvas ne dobije response | Audit + status provera |
| Invalid payload | Nedostaju obavezna polja | Kontrolisana greška bez rezervacije |
| Permission error | Flow nema pravo upisa | Critical error + admin alert |

---

## 16. Standardizovani response prema Canvas aplikaciji

### 16.1 Uspešan response

```json
{
  "success": true,
  "operation": "RegisterDocument",
  "correlationId": "GUID",
  "documentItemId": 123,
  "delovodniBroj": "Os.Del.Br. 101/2026",
  "reservationStatus": "Committed",
  "message": "Dokument je uspešno zaveden."
}
```

---

### 16.2 Greška kod konflikta

```json
{
  "success": false,
  "operation": "RegisterDocument",
  "correlationId": "GUID",
  "documentItemId": null,
  "delovodniBroj": null,
  "reservationStatus": "Failed",
  "message": "Dokument nije zaveden jer sistem nije mogao da rezerviše delovodni broj.",
  "error": {
    "code": "NUMBER_RESERVATION_FAILED",
    "details": "Maksimalan broj retry pokušaja je potrošen."
  }
}
```

---

### 16.3 Greška kod validacije

```json
{
  "success": false,
  "operation": "RegisterDocument",
  "correlationId": "GUID",
  "documentItemId": null,
  "delovodniBroj": null,
  "reservationStatus": "NotStarted",
  "message": "Dokument nije zaveden jer zahtev nije validan.",
  "error": {
    "code": "INVALID_PAYLOAD",
    "details": "Nedostaje obavezno polje."
  }
}
```

---

## 17. Audit log za numbering proces

Svaki pokušaj treba da se loguje.

Minimalni audit događaji:

| Event | Kada se upisuje |
|---|---|
| `RegistrationStarted` | Kada Canvas pokrene proces |
| `PayloadValidated` | Nakon uspešne validacije |
| `NumberReservationStarted` | Pre pokušaja rezervacije |
| `NumberReservationRetry` | Svaki retry |
| `NumberReserved` | Kada je broj rezervisan |
| `DocumentUploaded` | Kada je dokument sačuvan |
| `MetadataUpdated` | Kada je metadata upisan |
| `NumberCommitted` | Kada je broj potvrđen |
| `RegistrationCompleted` | Kada je proces završen |
| `RegistrationFailed` | Kada proces padne |

---

## 18. Kontrolne tačke za postojeće rešenje

Potrebno proveriti u postojećem solution-u:

| Pitanje | Status |
|---|---|
| Da li `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` koristi concurrency control? | NEPOZNATO |
| Da li postoji unique constraint na finalnom delovodnom broju? | NEPOZNATO |
| Da li `RezervisaniBrojevi.Title` ili drugo polje ima unique constraint? | Po XML-u `Title` nema unique constraint |
| Da li `RezervisaniBroj` ima unique constraint? | Po XML-u nema unique constraint |
| Da li postoji retry u flow-u? | NEPOZNATO |
| Da li postoji audit log? | NEPOZNATO |
| Da li se broj commit-uje nakon upload-a? | NEPOZNATO |
| Da li se neuspešne rezervacije čiste? | NEPOZNATO |

---

## 19. Zaključak

Za enterprise verziju DocCentral rešenja, numbering proces mora biti formalizovan kao backend servis.

Ključni princip:

```text
Delovodni broj se ne računa u aplikaciji.
Delovodni broj se traži od backend servisa.
Backend servis garantuje jedinstvenost.
SharePoint unique constraint služi kao poslednja linija odbrane.
Audit log dokazuje šta se dogodilo.
```

Najvažnija tehnička preporuka:

```text
Uvesti jedinstveni Numbering Service sa retry logikom, audit logom i unique constraint-om nad finalnim ključem delovodnog broja.
```

Bez toga sistem ostaje izložen race condition riziku kada više korisnika istovremeno zavodi dokumente.

---

## 20. Povezani dokumenti

```text
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
docs/10-known-issues-technical-debt.md
docs/11-enterprise-improvements.md
architecture/target-architecture.md
data-model/rezervisani-brojevi.md
backlog/enterprise-backlog.md
backlog/known-issues.md
```
