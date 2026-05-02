# 11. Enterprise unapređenja

## 1. Svrha dokumenta

Ovaj dokument definiše predlog enterprise unapređenja za buduću verziju rešenja **DocCentral v6.0 / DocCentral Next**.

Cilj nije samo tehničko sređivanje postojeće aplikacije, već prelazak na stabilniji, bezbedniji i skalabilniji model za elektronsku pisarnicu.

Dokument se zasniva na do sada poznatim informacijama iz analize:

- aplikacija je **Canvas App**
- početni ekran je `scrHome`
- zavođenje dokumenta postoji kao posebna funkcionalnost
- pregled dokumenata se trenutno radi direktno kroz SharePoint listu
- upload dokumenta ide preko Power Automate flow-a `Upload Doc`
- generisanje / dodela delovodnog broja ide preko posebnog flow-a zbog race condition problema
- flow za dodelu delovodnog broja je `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- `AppConfig` lista sadrži šifarnike, konfiguracije i JSON podešavanja
- `EmailDocuments` biblioteka koristi se za posebnu funkcionalnost gde Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-ima
- `Exports` biblioteka koristi se za export šifarnika i konfiguracije iz `AppConfig`
- najvažniji enterprise zahtev je da više korisnika istovremeno može da zavodi dokumenta, ali da sistem nikada ne sme dozvoliti dupliranje delovodnog broja

---

## 2. Glavni enterprise prioriteti

Enterprise verzija rešenja treba da reši sledeće prioritete:

1. concurrency-safe zavođenje dokumenata
2. centralizovano generisanje delovodnog broja
3. jasna podela odgovornosti između Canvas App, Power Automate i SharePoint sloja
4. bolji security i permission model
5. bolji audit i traceability
6. bolji logging za flow izvršavanja
7. bolja konfiguracija kroz `AppConfig`
8. standardizovan data model
9. stabilniji upload i obrada dokumenata
10. priprema za budući razvoj kroz Cloud Code / Claude Code / AI-assisted development

---

## 3. Concurrency-safe generisanje delovodnog broja

### 3.1 Problem

Kod pisarnice je delovodni broj kritičan podatak.

Ako dva korisnika istovremeno zavode dokument, sistem mora garantovati da neće dobiti isti broj.

Ovo ne sme da zavisi od:

- lokalne kolekcije u Canvas aplikaciji
- poslednjeg pročitanog item-a iz SharePoint liste
- neblokirane logike u Power Apps formuli
- ručnog povećavanja broja bez kontrole
- pretpostavke da se dva korisnika neće poklopiti u istom trenutku

### 3.2 Enterprise pravilo

Konačni delovodni broj mora se generisati isključivo na server-side strani.

U trenutnom kontekstu to znači kroz Power Automate flow, a ne u Canvas aplikaciji.

Canvas aplikacija sme da:

- prikaže formu
- validira osnovna obavezna polja
- pozove flow
- dobije rezultat
- prikaže korisniku finalni delovodni broj ili grešku

Canvas aplikacija ne sme da:

- sama garantuje jedinstvenost delovodnog broja
- sama upisuje finalni broj bez potvrde backend logike
- računa broj samo na osnovu lokalno učitanih podataka
- koristi `Max(...) + 1` kao finalni enterprise mehanizam

### 3.3 Predloženi model

Preporučeni model:

1. Canvas App šalje zahtev za zavođenje.
2. Flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` ili njegov refaktorisani naslednik preuzima zahtev.
3. Flow pokušava da rezerviše sledeći broj u kontrolisanoj transakciji.
4. Flow upisuje rezervaciju u `RezervisaniBrojevi`.
5. Flow proverava da broj nije već iskorišćen.
6. Flow kreira ili ažurira dokument / predmet.
7. Flow vraća Canvas aplikaciji:
   - status
   - finalni delovodni broj
   - ID kreiranog item-a
   - poruku za korisnika
   - tehnički correlation ID

### 3.4 Obavezne zaštite

Nova verzija mora imati:

- atomic ili optimistic locking mehanizam
- retry logiku
- unique constraint na finalnom delovodnom broju
- audit log rezervacije
- status rezervacije
- kontrolu neuspešne rezervacije
- kontrolu napuštene rezervacije
- correlation ID po pokušaju
- jasno korisničko obaveštenje ako rezervacija ne uspe

---

## 4. Refaktorisanje flow-a za delovodni broj

### 4.1 Postojeći flow

Poznat naziv flow-a:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Poznata namena:

```text
Poseban flow za dodelu / dodavanje delovodnog broja novom dokumentu iz Power Apps aplikacije.
```

Razlog postojanja:

```text
Sprečavanje race condition problema kada više korisnika istovremeno zavodi dokumenta.
```

### 4.2 Predlog unapređenja flow-a

Flow treba podeliti na logičke sekcije:

1. `Initialize request`
2. `Validate input`
3. `Create correlation ID`
4. `Reserve workbook number`
5. `Validate reservation`
6. `Create / update document`
7. `Finalize reservation`
8. `Write audit log`
9. `Return response to Power Apps`
10. `Handle errors`

### 4.3 Predloženi response prema Canvas App

Flow treba uvek da vrati standardizovan JSON:

```json
{
  "success": true,
  "status": "ReservedAndCreated",
  "message": "Dokument je uspešno zaveden.",
  "delovodniBroj": "Os.Del.Br. 123/2026",
  "reservedNumber": 123,
  "documentItemId": 1001,
  "correlationId": "GUID",
  "errorCode": null,
  "errorDetails": null
}
```

U slučaju greške:

```json
{
  "success": false,
  "status": "Failed",
  "message": "Dokument nije zaveden. Pokušajte ponovo ili kontaktirajte administratora.",
  "delovodniBroj": null,
  "reservedNumber": null,
  "documentItemId": null,
  "correlationId": "GUID",
  "errorCode": "NUMBER_RESERVATION_FAILED",
  "errorDetails": "Technical details for admin log"
}
```

---

## 5. Unapređenje `RezervisaniBrojevi` liste

### 5.1 Trenutno poznata polja

Poznata polja:

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

### 5.2 Predložena nova polja

Za enterprise model preporučuje se dodavanje sledećih polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Godina` | Number | Godina za koju se broj rezerviše |
| `TipDokumenta` | Choice/Text | Tip dokumenta ili knjige |
| `Prefix` | Text | Prefix delovodnog broja |
| `FullDelovodniBroj` | Text | Kompletan delovodni broj |
| `ReservationStatus` | Choice | Reserved, Used, Failed, Cancelled, Expired |
| `ReservedByEmail` | Text | Korisnik koji je pokrenuo rezervaciju |
| `ReservedForItemId` | Number | ID dokumenta / predmeta |
| `CorrelationId` | Text | Veza sa flow run-om |
| `RequestPayload` | Multiple lines of text | Ulazni JSON |
| `ResponsePayload` | Multiple lines of text | Izlazni JSON |
| `ErrorDetails` | Multiple lines of text | Tehnički detalji greške |

### 5.3 Unique constraint

Preporuka:

- `FullDelovodniBroj` treba da bude unique
- ako se broj vodi odvojeno po godini i tipu, onda treba obezbediti logičku jedinstvenost kombinacije:
  - `Godina`
  - `TipDokumenta`
  - `RezervisaniBroj`

Pošto SharePoint ne podržava composite unique constraint direktno, preporuka je da se uvede tekstualno polje:

```text
UniqueReservationKey
```

Primer vrednosti:

```text
2026|OS|123
```

Na tom polju treba uključiti unique constraint.

---

## 6. Unapređenje `AppConfig` modela

### 6.1 Trenutna namena

`AppConfig` sadrži:

- šifarnike
- konfiguracije
- JSON podešavanja
- podatke za export konfiguracije

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| `Title` | Title | Single line of text |
| `Config` | Config | Multiple lines of text |
| `ColumnHeader` | ColumnHeader | Multiple lines of text |
| `ID` | ID | Counter |
| `Created` | Created | Date and Time |
| `Modified` | Modified | Date and Time |

### 6.2 Predlog standardizacije

Preporučena struktura `AppConfig` item-a:

| Polje | Primer | Namena |
|---|---|---|
| `Title` | `DocumentTypes` | Ključ konfiguracije |
| `Config` | JSON array/object | Glavni sadržaj konfiguracije |
| `ColumnHeader` | JSON array/object | Header definicija za export/prikaz |
| `IsActive` | true/false | Da li je konfiguracija aktivna |
| `ConfigVersion` | 1.0.0 | Verzija konfiguracije |
| `Environment` | Dev/Test/Prod | Okruženje |
| `Description` | Text | Opis konfiguracije |

### 6.3 Validacija JSON-a

Preporuka:

- svaki `Config` mora biti validan JSON
- flow ili admin ekran treba da validira JSON pre upisa
- nevalidan JSON ne sme da bude sačuvan kao aktivna konfiguracija
- export iz `AppConfig` treba da sadrži verziju i datum exporta

---

## 7. Unapređenje upload procesa

### 7.1 Postojeće stanje

Poznato:

- upload dokumenta ide preko Power Automate flow-a `Upload Doc`
- dokumenti se čuvaju u SharePoint document library
- `Shared Documents` sadrži polja:
  - `DelovodniBroj`
  - `EdokumentID`
  - `Edokument`
  - `Attachment`

### 7.2 Predlog unapređenja

Upload proces treba da ima:

1. standardizovan input JSON
2. proveru obaveznih metapodataka
3. proveru tipa fajla
4. proveru veličine fajla
5. upis dokumenta
6. upis metapodataka
7. povezivanje sa delovodnim brojem
8. audit log
9. standardizovan response prema Canvas App

### 7.3 Predloženi response iz `Upload Doc` flow-a

```json
{
  "success": true,
  "status": "Uploaded",
  "message": "Dokument je uspešno uploadovan.",
  "library": "Shared Documents",
  "fileName": "document.pdf",
  "fileUrl": "https://...",
  "itemId": 1001,
  "delovodniBroj": "Os.Del.Br. 123/2026",
  "correlationId": "GUID"
}
```

---

## 8. Email intake unapređenje

### 8.1 Postojeće stanje

Poznato:

- `EmailDocuments` biblioteka koristi se za funkcionalnost gde Power Automate čita shared mailbox
- flow zavodi dokumente iz mailova sa attachment-ima
- poznata polja:
  - `PosiljalacEmail`
  - `Posiljalac`

### 8.2 Enterprise preporuke

Email intake proces treba da ima:

- kontrolu duplih email poruka
- kontrolu duplih attachment-a
- upis originalnog sender-a
- upis message ID-ja
- upis subject-a
- upis datuma prijema
- status obrade
- error log
- mogućnost retry obrade
- vezu sa finalnim delovodnim brojem
- vezu sa originalnim fajlom

### 8.3 Predložena nova polja za `EmailDocuments`

| Internal name | Tip | Namena |
|---|---|---|
| `EmailMessageId` | Text | Jedinstveni ID email poruke |
| `EmailSubject` | Text | Subject email-a |
| `ReceivedDateTime` | Date and Time | Datum prijema |
| `AttachmentName` | Text | Naziv attachment-a |
| `AttachmentHash` | Text | Kontrola duplikata |
| `ProcessingStatus` | Choice | New, Processed, Failed, Duplicate |
| `ProcessingError` | Multiple lines of text | Greška obrade |
| `DelovodniBroj` | Text | Finalni delovodni broj |

---

## 9. Security i permission unapređenja

### 9.1 Princip

Enterprise verzija treba da koristi princip najmanjih privilegija.

Korisnici ne treba direktno da imaju veća prava nad SharePoint listama i bibliotekama nego što je neophodno.

### 9.2 Preporučeni model

Preporuka:

- korisnici imaju read-only pristup relevantnim SharePoint podacima
- sve Create/Edit/Delete operacije idu preko Power Automate
- flow-ovi rade preko kontrolisanog service account-a
- service account ima potrebna prava za upis
- korisničke akcije se loguju sa stvarnim korisnikom koji je inicirao akciju
- admin funkcije su odvojene od korisničkih funkcija

### 9.3 Kritična napomena

Ako flow radi pod service account-om, audit mora posebno čuvati stvarnog inicijatora akcije.

Obavezna audit polja:

- `InitiatedByDisplayName`
- `InitiatedByEmail`
- `InitiatedFromApp`
- `CorrelationId`
- `FlowName`
- `FlowRunId`
- `ActionType`
- `TargetListOrLibrary`
- `TargetItemId`
- `Timestamp`

---

## 10. Logging i monitoring

### 10.1 Problem

Bez centralnog logovanja teško je analizirati:

- zašto je zavođenje puklo
- da li je broj rezervisan ali dokument nije kreiran
- da li je upload uspeo
- koji flow je napravio grešku
- koji korisnik je pokrenuo akciju
- da li postoji problem sa concurrency logikom

### 10.2 Predlog

Uvesti centralnu SharePoint listu:

```text
FlowExecutionLog
```

ili, ako licenciranje i arhitektura dozvoljavaju, Dataverse tabelu.

### 10.3 Predložena polja

| Internal name | Tip | Namena |
|---|---|---|
| `Title` | Text | Kratak opis log zapisa |
| `CorrelationId` | Text | Jedinstveni ID procesa |
| `FlowName` | Text | Naziv flow-a |
| `FlowRunId` | Text | ID izvršavanja |
| `ActionType` | Text | Tip akcije |
| `Status` | Choice | Started, Success, Failed |
| `StartedAt` | Date and Time | Početak |
| `FinishedAt` | Date and Time | Kraj |
| `DurationMs` | Number | Trajanje |
| `InitiatedByEmail` | Text | Korisnik |
| `TargetItemId` | Number | ID stavke |
| `RequestJson` | Multiple lines of text | Ulaz |
| `ResponseJson` | Multiple lines of text | Izlaz |
| `ErrorJson` | Multiple lines of text | Greška |

---

## 11. Canvas App unapređenja

### 11.1 Poznato stanje

Poznato:

- aplikacija je Canvas App
- početni ekran je `scrHome`
- postoji funkcionalnost za zavođenje dokumenta
- pregled dokumenata se ne radi kroz poseban ekran, već direktno u SharePoint listi

### 11.2 Preporučena unapređenja

Canvas App treba unaprediti tako da:

- bude potpuno responzivna
- ima jasnu navigaciju
- razdvaja korisničke i admin funkcije
- ne sadrži kritičnu poslovnu logiku za delovodni broj
- koristi flow response kao izvor istine
- prikazuje korisniku jasne poruke
- ima loading/spinner stanja
- ima kontrolu grešaka
- ima centralizovane komponente za poruke
- ima jasnu validaciju forme
- ima role-based prikaz funkcija

### 11.3 Predloženi ekrani za novu verziju

| Ekran | Namena |
|---|---|
| `scrHome` | Početni ekran / dashboard |
| `scrRegisterDocument` | Zavođenje dokumenta |
| `scrUploadDocument` | Upload dokumenta |
| `scrEmailDocuments` | Pregled email dokumenata |
| `scrAppConfigAdmin` | Administracija konfiguracije |
| `scrExportConfig` | Export šifarnika i konfiguracije |
| `scrLogs` | Pregled logova |
| `scrErrorDetails` | Detalji greške |

---

## 12. SharePoint data model unapređenja

### 12.1 Indeksi

Preporučeni indeksi:

- `DelovodniBroj`
- `EdokumentID`
- `Edokument`
- `Attachment`
- `Created`
- `Modified`
- `PosiljalacEmail`
- `ReservationStatus`
- `CorrelationId`
- `UniqueReservationKey`

### 12.2 Naming convention

Preporuka:

- koristiti stabilna interna imena bez razmaka i specijalnih karaktera
- koristiti engleska ili srpska imena dosledno, ne mešati bez pravila
- izbegavati promenu display name-a nakon kreiranja ako može da zbuni administratore
- dokumentovati svako polje u data model dokumentima

### 12.3 Status polja

Za svaku ključnu listu / biblioteku treba uvesti statusna polja.

Primeri:

```text
Draft
Reserved
Uploaded
Registered
Processed
Failed
Cancelled
Archived
```

---

## 13. Export unapređenja

### 13.1 Postojeće stanje

Poznato:

- `Exports` biblioteka koristi se za export šifarnika i konfiguracije iz `AppConfig`

### 13.2 Preporuke

Export treba da sadrži:

- datum exporta
- korisnika koji je pokrenuo export
- verziju konfiguracije
- naziv konfiguracije
- okruženje
- JSON payload
- status exporta
- link ka generisanom fajlu

### 13.3 Predlog naziva fajla

```text
AppConfig_Export_Prod_2026-04-30_165900.json
```

---

## 14. Administracija i governance

### 14.1 Predlog admin funkcija

Admin deo aplikacije treba da omogući:

- pregled `AppConfig`
- validaciju JSON konfiguracije
- export konfiguracije
- aktivaciju/deaktivaciju konfiguracije
- pregled rezervisanih brojeva
- pregled neuspelih rezervacija
- pregled email intake grešaka
- pregled flow logova
- retry neuspelih procesa

### 14.2 Role-based access

Predložene role:

| Rola | Namena |
|---|---|
| `DocCentral.Reader` | Pregled podataka |
| `DocCentral.Clerk` | Zavođenje dokumenata |
| `DocCentral.Manager` | Nadzor i odobravanje |
| `DocCentral.Admin` | Administracija konfiguracije |
| `DocCentral.ServiceAccount` | Tehničko izvršavanje flow-ova |

---

## 15. Predlog target arhitekture

```text
Canvas App
  |
  | korisnik popunjava formu i pokreće akciju
  v
Power Automate API layer
  |
  | validacija, rezervacija broja, upload, audit
  v
SharePoint Online data/storage layer
  |
  | liste, biblioteke, konfiguracija, logovi
  v
Audit / Export / Reporting
```

### 15.1 Ključni princip

Power Apps je UI sloj.

Power Automate je poslovni i integracioni sloj.

SharePoint je data i document storage sloj.

---

## 16. Prioriteti implementacije

### Prioritet 1 — kritično

- concurrency-safe generisanje delovodnog broja
- unique key / unique constraint
- retry logika
- audit log
- standardizovan response iz flow-a

### Prioritet 2 — visoko

- refaktorisanje `Upload Doc` flow-a
- standardizacija `AppConfig`
- centralni logging
- role-based permission model
- admin pregled grešaka

### Prioritet 3 — srednje

- responzivni UI
- bolji dashboard na `scrHome`
- pregled email dokumenata
- export istorija
- bolji monitoring

### Prioritet 4 — kasnije

- Dataverse migracija
- Application Insights
- Power BI monitoring
- automatizovani deployment pipeline
- solution ALM standardizacija

---

## 17. Otvorena pitanja

1. Da li `RezervisaniBrojevi` trenutno ima unique constraint na nekom polju?
2. Da li `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` trenutno koristi SharePoint item ETag / optimistic concurrency?
3. Da li `Upload Doc` vraća standardizovan JSON response ili samo osnovni output?
4. Da li postoji posebna lista za audit log?
5. Da li postoji posebna lista za flow logove?
6. Da li `AppConfig.Config` trenutno uvek sadrži validan JSON?
7. Da li postoje različite konfiguracije za Dev/Test/Prod?
8. Da li se email intake dokumenti automatski prebacuju iz `EmailDocuments` u `Shared Documents`?
9. Da li korisnici trenutno imaju read-only ili edit prava nad SharePoint listama?
10. Da li postoji plan za Dataverse ili SharePoint ostaje glavni data layer?

---

## 18. Zaključak

DocCentral v6.0 već ima važne elemente enterprise arhitekture:

- Canvas App kao korisnički sloj
- Power Automate kao backend procesni sloj
- SharePoint Online kao data i document storage sloj
- `AppConfig` kao centralnu konfiguracionu listu
- `RezervisaniBrojevi` kao osnovu za kontrolu delovodnih brojeva
- posebne biblioteke za dokumente, email dokumente i exporte

Najkritičnija oblast za buduću verziju je **sigurno generisanje delovodnog broja bez duplikata u situaciji paralelnog rada više korisnika**.

Nova verzija mora tretirati zavođenje dokumenta kao backend-controlled proces, sa jasnim atomic/locking mehanizmom, retry logikom, audit tragom i unique constraint zaštitom.

Ovaj dokument predstavlja osnovu za naredni dokument:

```text
12-cloud-code-development-brief.md
```
