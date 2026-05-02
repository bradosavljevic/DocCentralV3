# Target Architecture — DocCentral v6.0 / Next Enterprise Version

## 1. Svrha dokumenta

Ovaj dokument opisuje ciljnu arhitekturu za novu enterprise verziju rešenja **DocCentral**.

Ciljna arhitektura je izvedena iz postojeće arhitekture DocCentral v6.0, poznatih SharePoint komponenti, potvrđenih funkcionalnosti i ključnog poslovnog zahteva:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovaj zahtev ima najviši prioritet i mora biti tretiran kao centralni arhitektonski princip.

---

## 2. Ciljevi ciljne arhitekture

Nova verzija treba da obezbedi:

- stabilno zavođenje dokumenata za više korisnika istovremeno
- garantovano jedinstven delovodni broj
- razdvajanje UI logike od poslovne logike
- backend kontrolu nad Create / Edit / Delete operacijama
- centralnu konfiguraciju kroz `AppConfig`
- audit log za sve kritične operacije
- bolje upravljanje greškama
- bolju skalabilnost
- spremnost za dalji razvoj kroz Cloud Code / Claude Code / AI-assisted development
- jasnu tehničku dokumentaciju pogodnu za GitHub

---

## 3. Arhitektonski principi

### 3.1 Canvas App je UI sloj

Power Apps Canvas aplikacija treba da bude odgovorna za:

- prikaz forme
- unos podataka
- osnovnu UI validaciju
- prikaz grešaka korisniku
- pozivanje backend flow-ova
- prikaz rezultata operacije

Canvas aplikacija ne sme biti odgovorna za:

- finalno generisanje delovodnog broja
- proveru jedinstvenosti delovodnog broja
- direktan kritičan upis u SharePoint bez backend kontrole
- rešavanje race condition problema
- sigurnosnu validaciju prava
- finalnu poslovnu odluku

---

### 3.2 Power Automate je business service sloj

Power Automate treba da bude centralno mesto za:

- kreiranje dokumenata
- upload dokumenata
- dodelu delovodnog broja
- ažuriranje SharePoint metadata
- validaciju poslovnih pravila
- audit log
- kontrolisano vraćanje rezultata u Canvas aplikaciju
- obradu email dokumenata
- export konfiguracije

---

### 3.3 SharePoint Online je data i document storage sloj

SharePoint Online treba da ostane storage sloj za:

- konfiguraciju
- dokumente
- email dokumente
- export fajlove
- rezervisane brojeve
- audit logove
- poslovne liste i šifarnike

Međutim, SharePoint ne treba koristiti kao pasivni storage bez kontrole. Potrebno je uvesti jasna pravila za:

- unique constraints
- indeksiranje kolona
- permission model
- audit trail
- lifecycle podataka
- error log
- backup / restore konfiguracije

---

## 4. Ciljna arhitektura na visokom nivou

```text
Korisnik
   |
   v
Power Apps Canvas App
   |
   | 1. Validira unos na UI nivou
   | 2. Poziva backend flow
   v
Power Automate Business Layer
   |
   | 1. Validira zahtev
   | 2. Rezerviše / generiše delovodni broj
   | 3. Uploaduje dokument
   | 4. Upisuje metadata
   | 5. Upisuje audit log
   | 6. Vraća rezultat Canvas aplikaciji
   v
SharePoint Online
   |
   | Liste + biblioteke + konfiguracija + audit
   v
DocCentral evidencija dokumenata
```

---

## 5. Predloženi logički slojevi

### 5.1 UI Layer

Komponenta:

```text
Power Apps Canvas App
```

Odgovornosti:

- `scrHome` kao početni ekran
- ekran za zavođenje dokumenta
- prikaz konfiguracije iz `AppConfig`
- pozivanje flow-ova
- prikaz statusa obrade
- prikaz kontrolisanih grešaka
- osnovna validacija obaveznih polja

Ne sme:

- finalno generisati delovodni broj
- oslanjati se na lokalnu kolekciju kao izvor istine za sledeći broj
- direktno menjati kritične SharePoint zapise ako korisnici treba da imaju read-only prava

---

### 5.2 Business Flow Layer

Komponenta:

```text
Power Automate
```

Odgovornosti:

- centralizovana poslovna pravila
- kreiranje i ažuriranje dokumenata
- numbering service
- upload service
- email intake service
- export service
- audit service
- error handling
- kontrolisani response prema Canvas aplikaciji

Predloženi flow-ovi / servisne grupe:

| Servis | Namena |
|---|---|
| Document Upload Service | Upload dokumenta i metadata |
| Numbering Service | Generisanje i rezervacija delovodnog broja |
| Document Registration Service | Finalno zavođenje dokumenta |
| Email Intake Service | Obrada shared mailbox dokumenata |
| Export Service | Export `AppConfig` i šifarnika |
| Audit Service | Upis audit logova |
| Error Handling Service | Centralna obrada grešaka |

---

### 5.3 Numbering Service

Ovo je najkritičniji deo sistema.

Cilj:

```text
Nikada ne smeju postojati dva dokumenta sa istim delovodnim brojem.
```

Numbering service mora da obezbedi:

- centralizovano generisanje broja
- atomic update ili optimistic locking
- retry logiku
- unique constraint
- audit log
- kontrolisan response
- rollback ili označavanje neuspešnih rezervacija
- jasnu vezu između rezervacije i finalnog dokumenta

---

### 5.4 Configuration Layer

Komponenta:

```text
SharePoint lista AppConfig
```

Namena:

- šifarnici
- JSON konfiguracije
- podešavanja aplikacije
- kolone / header konfiguracija
- export konfiguracije
- potencijalno prevodi i UI podešavanja

Preporuke:

- uvesti `ConfigType`
- uvesti `ConfigKey`
- uvesti `ConfigVersion`
- uvesti `IsActive`
- uvesti `ValidFrom`
- uvesti `ValidTo`
- uvesti JSON validation flow
- uvesti approval za promenu produkcione konfiguracije

---

### 5.5 Document Storage Layer

Komponente:

```text
Shared Documents
EmailDocuments
Exports
```

Namena:

| Biblioteka | Namena |
|---|---|
| `Shared Documents` | Glavni dokumenti |
| `EmailDocuments` | Dokumenti iz email procesa |
| `Exports` | Export konfiguracije i šifarnika |

Preporuke:

- standardizovati metadata
- indeksirati ključne kolone
- proveriti unique constraint za `DelovodniBroj`
- razdvojiti ulazne, obrađene i arhivirane dokumente ako obim raste
- uvesti lifecycle pravila
- uvesti audit za svaku promenu metadata

---

### 5.6 Audit and Logging Layer

Predložena nova SharePoint lista:

```text
AuditLog
```

Minimalna polja:

| Polje | Tip | Namena |
|---|---|---|
| `Title` | Text | Kratak naziv događaja |
| `CorrelationId` | Text | ID operacije kroz flow-ove |
| `OperationType` | Choice/Text | Tip operacije |
| `Source` | Text | Canvas App / Flow / Email |
| `UserEmail` | Text | Korisnik koji je pokrenuo operaciju |
| `DocumentId` | Number/Text | ID dokumenta ili stavke |
| `DelovodniBroj` | Text | Dodeljeni broj |
| `Status` | Choice/Text | Success / Failed / Retry / Reserved |
| `ErrorMessage` | Multiple lines | Poruka greške |
| `Payload` | Multiple lines | JSON zahtev / response |
| `CreatedAt` | DateTime | Vreme događaja |

Namena:

- praćenje procesa zavođenja
- dijagnostika race condition problema
- forenzička analiza grešaka
- tehnička podrška
- enterprise audit trail

---

## 6. Predložena ciljna struktura Power Automate procesa

### 6.1 Ručno zavođenje dokumenta

```text
Canvas App
   |
   | poziva
   v
Document Registration Flow
   |
   | validira zahtev
   v
Numbering Service Flow
   |
   | rezerviše delovodni broj
   v
Upload Document Flow
   |
   | uploaduje fajl
   v
Update Metadata Flow
   |
   | upisuje metadata i delovodni broj
   v
Audit Log Flow
   |
   | upisuje uspeh ili grešku
   v
Response to Canvas App
```

---

### 6.2 Email intake proces

```text
Shared Mailbox
   |
   v
Email Intake Flow
   |
   | čita emailove sa attachmentima
   v
Validate Email / Attachment
   |
   v
Save to EmailDocuments
   |
   v
Register Document / Assign Number
   |
   v
Move / mark email as processed
   |
   v
Audit Log
```

Preporuke:

- obavezna deduplikacija
- obrada samo validnih attachment-a
- čuvanje MessageId vrednosti
- čuvanje pošiljaoca
- kontrola grešaka po attachment-u
- mogućnost reprocess operacije

---

### 6.3 Export proces

```text
Admin / Scheduled trigger
   |
   v
Export AppConfig Flow
   |
   | čita AppConfig
   v
Generate JSON / Excel / CSV
   |
   v
Save to Exports library
   |
   v
Audit Log
```

Preporuke:

- export mora biti verzionisan
- export mora imati datum i verziju
- export mora sadržati informaciju ko ga je pokrenuo
- poželjno je imati i import/restore proces

---

## 7. Concurrency-safe numbering model

### 7.1 Obavezni uslovi

Numbering model mora da ispuni sledeće uslove:

- dva korisnika mogu istovremeno pokrenuti zavođenje
- samo jedan korisnik može dobiti određeni broj
- drugi korisnik mora dobiti sledeći slobodan broj
- sistem mora znati koji broj je rezervisan, iskorišćen, poništen ili neuspešan
- delovodni broj mora biti jedinstven i proverljiv
- greška ne sme ostaviti sistem u nejasnom stanju

---

### 7.2 Predloženi statusi rezervacije

Za listu `RezervisaniBrojevi` ili novu listu `NumberReservations` preporučuju se statusi:

| Status | Značenje |
|---|---|
| `Reserved` | Broj je rezervisan, ali dokument još nije finalno kreiran |
| `Committed` | Broj je iskorišćen i vezan za dokument |
| `Cancelled` | Rezervacija je poništena |
| `Expired` | Rezervacija je istekla |
| `Failed` | Proces nije uspeo |

---

### 7.3 Minimalna polja za novu numbering listu

Predložena lista:

```text
NumberReservations
```

Polja:

| Polje | Tip | Namena |
|---|---|---|
| `Title` | Text | Delovodni broj |
| `NumberValue` | Number | Numerički deo broja |
| `NumberPrefix` | Text | Prefiks |
| `NumberYear` | Number | Godina |
| `ReservationStatus` | Choice | Reserved / Committed / Cancelled / Expired / Failed |
| `ReservedByEmail` | Text | Korisnik koji je rezervisao broj |
| `ReservedAt` | DateTime | Vreme rezervacije |
| `CommittedAt` | DateTime | Vreme finalnog upisa |
| `DocumentItemId` | Number | ID dokumenta / stavke |
| `CorrelationId` | Text | ID transakcije |
| `ErrorMessage` | Multiple lines | Greška ako postoji |

Obavezno:

```text
Title ili finalni DelovodniBroj mora imati unique constraint.
```

---

## 8. Security model u ciljnoj arhitekturi

Preporučeni model:

```text
Korisnici: Read / Limited access nad SharePoint podacima
Power Automate service account: Write / Edit prava
Administratori: Full Control / Owner prava
```

Cilj:

- korisnici ne menjaju direktno kritične podatke
- sve promene idu kroz kontrolisane flow-ove
- audit je centralizovan
- smanjuje se rizik od ručnih izmena
- smanjuje se rizik od zaobilaženja poslovnih pravila

Potrebno potvrditi za postojeće okruženje:

- da li postoji service account
- ko je vlasnik flow-ova
- koja prava imaju korisnici nad SharePoint listama
- da li flow-ovi rade sa elevated permissions
- da li postoje posebne grupe za korisnike, administratore i servisne naloge

---

## 9. Predložena ciljna SharePoint struktura

### 9.1 Postojeće komponente koje ostaju

```text
AppConfig
RezervisaniBrojevi
Shared Documents
EmailDocuments
Exports
```

---

### 9.2 Predložene nove liste

| Lista | Namena |
|---|---|
| `AuditLog` | Centralni audit svih kritičnih operacija |
| `ErrorLog` | Tehničke greške flow-ova |
| `NumberReservations` | Enterprise rezervacija delovodnih brojeva |
| `AppConfigVersions` | Verzije konfiguracije |
| `EmailProcessingLog` | Log obrade emailova |
| `DocumentOperationsLog` | Log operacija nad dokumentima |

Napomena:

Neke od ovih lista mogu biti spojene ako je cilj jednostavnija arhitektura, ali za enterprise rešenje je bolje jasno razdvajanje.

---

## 10. Predloženi response model prema Canvas aplikaciji

Svi backend flow-ovi koje poziva Canvas aplikacija treba da vraćaju standardizovan JSON response.

Primer:

```json
{
  "success": true,
  "correlationId": "GUID",
  "operation": "RegisterDocument",
  "documentItemId": 123,
  "delovodniBroj": "Os.Del.Br. 15/2026",
  "message": "Dokument je uspešno zaveden.",
  "error": null
}
```

Primer greške:

```json
{
  "success": false,
  "correlationId": "GUID",
  "operation": "RegisterDocument",
  "documentItemId": null,
  "delovodniBroj": null,
  "message": "Dokument nije zaveden.",
  "error": {
    "code": "NUMBERING_CONFLICT",
    "details": "Delovodni broj nije mogao biti rezervisan nakon maksimalnog broja pokušaja."
  }
}
```

Prednosti:

- lakše prikazivanje poruka u Canvas aplikaciji
- bolja dijagnostika
- standardizovan error handling
- lakša integracija sa budućim servisima

---

## 11. Target architecture dijagram

```text
+--------------------------------------------------+
| User                                             |
+--------------------------------------------------+
                       |
                       v
+--------------------------------------------------+
| Power Apps Canvas App                            |
| - scrHome                                        |
| - Document registration screen                   |
| - UI validation                                  |
| - AppConfig-driven UI                            |
+--------------------------------------------------+
                       |
                       v
+--------------------------------------------------+
| Power Automate Business Layer                    |
|                                                  |
| - Document Registration Service                  |
| - Upload Doc Service                             |
| - Numbering Service                              |
| - Email Intake Service                           |
| - Export Service                                 |
| - Audit / Error Logging                          |
+--------------------------------------------------+
                       |
                       v
+--------------------------------------------------+
| SharePoint Online                                |
|                                                  |
| Lists:                                           |
| - AppConfig                                      |
| - NumberReservations / RezervisaniBrojevi        |
| - AuditLog                                       |
| - ErrorLog                                       |
|                                                  |
| Libraries:                                       |
| - Shared Documents                               |
| - EmailDocuments                                 |
| - Exports                                        |
+--------------------------------------------------+
```

---

## 12. Migracioni pristup

Preporučeni pristup nije potpuni rewrite odjednom, već kontrolisana evolucija.

### Faza 1 — Dokumentacija i stabilizacija

- dokumentovati postojeće liste
- dokumentovati postojeće flow-ove
- dokumentovati Canvas ekrane
- identifikovati sve kritične tačke
- potvrditi trenutni numbering mehanizam

### Faza 2 — Numbering hardening

- proveriti `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- uvesti unique constraint
- uvesti retry logiku
- uvesti audit log
- testirati paralelno zavođenje

### Faza 3 — Security hardening

- uvesti service account model
- ograničiti direktna prava korisnika
- prebaciti Create / Edit / Delete kroz flow-ove
- dokumentovati permission model

### Faza 4 — AppConfig hardening

- uvesti verzionisanje konfiguracije
- validirati JSON
- napraviti export / import strategiju
- uvesti approval za promene konfiguracije

### Faza 5 — Nova enterprise verzija

- responzivna Canvas aplikacija
- bolja komponentizacija
- standardizovani flow response-i
- centralizovani audit
- bolji error handling
- priprema za integracije

---

## 13. Test scenariji za ciljnu arhitekturu

### 13.1 Paralelno zavođenje

Test:

- korisnik A i korisnik B istovremeno zavode dokument
- oba procesa pozivaju numbering service
- sistem mora vratiti različite delovodne brojeve

Očekivani rezultat:

```text
Nema duplog delovodnog broja.
```

---

### 13.2 Greška tokom upload-a nakon rezervacije broja

Test:

- broj je rezervisan
- upload dokumenta ne uspe
- sistem mora označiti rezervaciju kao `Failed`, `Cancelled` ili `Expired`

Očekivani rezultat:

```text
Sistem ne ostaje u nejasnom stanju.
```

---

### 13.3 Greška tokom dodele broja

Test:

- numbering service ne uspe nakon maksimalnog broja retry pokušaja
- Canvas aplikacija dobija kontrolisanu grešku

Očekivani rezultat:

```text
Korisnik vidi jasnu poruku, a greška je upisana u log.
```

---

### 13.4 Izmena AppConfig JSON-a

Test:

- administrator unese nevalidan JSON
- sistem validira konfiguraciju pre produkcione upotrebe

Očekivani rezultat:

```text
Nevalidna konfiguracija se ne aktivira.
```

---

## 14. Zaključak

Ciljna arhitektura treba da zadrži prednosti trenutnog Power Platform pristupa, ali da uvede enterprise kontrolu nad kritičnim procesima.

Najvažnija promena je formalizacija backend sloja i numbering service-a.

Ključni zaključak:

```text
Canvas aplikacija prikazuje i pokreće proces.
Power Automate kontroliše poslovnu logiku.
SharePoint čuva podatke.
Numbering service garantuje jedinstven delovodni broj.
Audit log dokazuje šta se desilo.
```

Bez ovih principa sistem može raditi funkcionalno, ali nije dovoljno robustan za enterprise scenario u kojem više korisnika istovremeno zavodi dokumente.

---

## 15. Povezani dokumenti

```text
docs/02-current-architecture.md
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
docs/11-enterprise-improvements.md
architecture/current-architecture.md
architecture/numbering-sequence.md
data-model/rezervisani-brojevi.md
backlog/enterprise-backlog.md
```
