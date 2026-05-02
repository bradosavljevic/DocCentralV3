# Known Issues & Technical Debt

## 1. Svrha dokumenta

Ovaj dokument evidentira poznate probleme, rizike i tehnički dug u postojećem DocCentral v6.0 rešenju.

Dokument služi kao:

- backlog za tehnička unapređenja
- osnova za planiranje nove enterprise verzije
- input za Cloud Code / Claude Code razvoj
- kontrolna lista za arhitektonske odluke
- pregled rizika koje treba rešiti pre produkcionog proširenja

---

## 2. Najkritičniji poznati rizik

### 2.1 Rizik duplog delovodnog broja

Najvažniji rizik u rešenju je mogućnost da dva korisnika istovremeno zavode dokument i potencijalno dobiju isti delovodni broj.

Status:

```text
KRITIČNO
```

Poznato:

- korisnici moraju moći istovremeno da zavode dokumenta
- sistem nikada ne sme dozvoliti isti delovodni broj za dva dokumenta
- zbog race condition problema postoji poseban flow:
  - `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- upload dokumenta ide preko flow-a:
  - `Upload Doc`

Rizik:

```text
Ako se delovodni broj računa u Canvas aplikaciji ili na osnovu poslednjeg pročitanog broja bez atomic kontrole, postoji rizik od duplikata.
```

Obavezna korekcija:

- finalna dodela delovodnog broja mora biti server-side
- mora postojati concurrency-safe mehanizam
- mora postojati unique constraint
- mora postojati retry logika
- mora postojati audit log
- mora postojati kontrolisan response prema Canvas aplikaciji

Prioritet:

```text
P0 / Critical
```

---

## 3. Tehnički dug: Canvas aplikacija

### 3.1 Kritična logika ne sme biti u Canvas aplikaciji

Rizik:

Canvas aplikacija nije pogodna da bude izvor istine za kritične poslovne odluke kao što je finalna dodela delovodnog broja.

Pravilo:

```text
Canvas App sme da inicira proces, ali ne sme da garantuje jedinstvenost broja.
```

Preporuka:

- Canvas aplikacija šalje zahtev Power Automate flow-u
- flow validira podatke
- flow rezerviše broj
- flow upisuje dokument
- flow vraća rezultat aplikaciji

Prioritet:

```text
P0 / Critical
```

---

### 3.2 Nepoznata kompletna struktura ekrana

Poznato:

- aplikacija je Canvas App
- početni ekran je `scrHome`
- postoji funkcionalnost za zavođenje dokumenta

Nepoznato:

- tačan naziv ekrana za zavođenje dokumenta
- tačan naziv ekrana za upload
- da li postoje komponente
- da li postoje globalne varijable za konfiguraciju
- da li postoji višejezičnost
- da li postoji centralizovano rukovanje greškama

Preporuka:

- eksportovati Canvas app
- analizirati `.msapp` / unpacked source
- dokumentovati ekrane, kontrole, kolekcije i flow pozive

Prioritet:

```text
P1 / High
```

---

## 4. Tehnički dug: Power Automate

### 4.1 Nepoznata unutrašnja logika flow-a za delovodni broj

Poznat flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Poznata namena:

```text
Dodela / dodavanje delovodnog broja novom dokumentu iz Power Apps aplikacije.
```

Nepoznato:

- da li koristi ETag
- da li koristi retry logiku
- da li koristi SharePoint unique constraint
- da li piše u `RezervisaniBrojevi`
- da li vraća standardizovan JSON response
- da li ima centralizovan error handling
- da li loguje correlation ID
- da li postoji rollback ako upload dokumenta ne uspe

Rizik:

```text
Ako flow samo pročita poslednji broj i poveća ga za 1, bez atomic kontrole, problem race condition-a nije potpuno rešen.
```

Preporuka:

- otvoriti flow
- eksportovati definiciju flow-a
- analizirati akcije, trigger, concurrency settings i retry policy
- dokumentovati tačan algoritam dodele broja

Prioritet:

```text
P0 / Critical
```

---

### 4.2 Nepoznata logika `Upload Doc` flow-a

Poznat flow:

```text
Upload Doc
```

Poznata namena:

```text
Upload dokumenta preko Power Automate flow-a.
```

Nepoznato:

- koji trigger koristi
- koji input prima iz Canvas App
- da li upload ide u `Shared Documents`
- da li update-uje metapodatke posle kreiranja fajla
- da li vraća item ID
- da li vraća file URL
- da li vraća greške u standardizovanom JSON formatu
- da li ima rollback ako kasniji korak ne uspe

Rizik:

```text
Ako upload i dodela broja nisu transakcijski povezani, može nastati dokument bez ispravnog statusa ili broj bez dokumenta.
```

Preporuka:

- uvesti correlation ID
- povezati upload i rezervaciju broja kroz isti proces ili jasan orchestration flow
- dodati status obrade
- dodati audit zapis

Prioritet:

```text
P1 / High
```

---

### 4.3 Nepoznat email intake flow

Poznato:

```text
Posebna funkcionalnost čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.
```

Nepoznato:

- naziv flow-a
- mailbox adresa
- da li obrađuje samo attachment-e ili i telo maila
- da li sprečava duplu obradu istog emaila
- da li čuva Message ID
- da li čuva pošiljaoca
- da li odmah zavodi dokument ili ga stavlja u staging
- da li koristi istu logiku delovodnog broja kao Canvas App

Rizik:

```text
Email intake može proizvesti duplikate ako nema Message ID / Attachment Hash / Processing Status kontrolu.
```

Preporuka:

- dodati `EmailMessageId`
- dodati `AttachmentHash`
- dodati `ProcessingStatus`
- dodati retry i dead-letter logiku
- koristiti isti server-side numbering servis kao Canvas App

Prioritet:

```text
P1 / High
```

---

### 4.4 Nepoznat export flow

Poznato:

```text
Export šifarnika i konfiguracije iz AppConfig liste.
```

Nepoznato:

- naziv flow-a
- format exporta
- da li exportuje JSON
- da li validira JSON pre exporta
- da li upisuje u `Exports`
- da li beleži verziju konfiguracije
- da li postoji audit

Preporuka:

- standardizovati export format
- dodati `ConfigVersion`
- dodati `ExportType`
- dodati `ExportedAt`
- dodati `ExportedByEmail`
- dodati `CorrelationId`

Prioritet:

```text
P2 / Medium
```

---

## 5. Tehnički dug: SharePoint data model

### 5.1 `RezervisaniBrojevi` nema potvrđen unique constraint

Poznata polja:

- `Title`
- `RezervisaniBroj`
- `DatumRezervacije`

Nepoznato:

- da li postoji unique constraint
- da li je `Title` jedinstven
- da li postoji složeni ključ po godini / tipu dokumenta / broju
- da li se čuva status rezervacije
- da li se čuva korisnik koji je rezervisao broj

Rizik:

```text
SharePoint Number polje samo po sebi ne garantuje jedinstvenost poslovnog broja u svim scenarijima.
```

Preporuka:

Dodati tekstualni unique key:

```text
UniqueReservationKey
```

Primer:

```text
2026|OS|123
```

Na tom polju uključiti unique constraint.

Prioritet:

```text
P0 / Critical
```

---

### 5.2 `Shared Documents` nema potvrđen unique constraint na `DelovodniBroj`

Poznato:

- biblioteka ima polje `DelovodniBroj`
- tip polja je `Single line of text`

Nepoznato:

- da li je `DelovodniBroj` indeksiran
- da li ima unique constraint
- da li se koristi kao finalni poslovni ključ
- da li se popunjava pre ili posle upload-a

Rizik:

```text
Ako `DelovodniBroj` nije unique, biblioteka može prihvatiti dva dokumenta sa istim poslovnim brojem.
```

Preporuka:

- indeksirati `DelovodniBroj`
- uključiti unique constraint ako poslovna pravila dozvoljavaju
- uskladiti sa `RezervisaniBrojevi`
- uvesti kontrolni audit između rezervacije i dokumenta

Prioritet:

```text
P0 / Critical
```

---

### 5.3 AppConfig sadrži konfiguraciju, ali sadržaj nije analiziran

Poznato:

- `AppConfig` ima polje `Config`
- `Config` je Multiple lines of text
- `ColumnHeader` je Multiple lines of text
- lista služi za šifarnike i konfiguracije

Nepoznato:

- struktura JSON-a
- verzionisanje konfiguracije
- validacija JSON-a
- da li postoje aktivne/neaktivne konfiguracije
- da li postoje konfiguracije po okruženju

Rizik:

```text
Bez validacije i verzionisanja, greška u JSON konfiguraciji može oboriti deo aplikacije.
```

Preporuka:

- definisati JSON schema
- dodati `ConfigVersion`
- dodati `IsActive`
- dodati `Environment`
- dodati export/import proceduru
- dodati validaciju pre snimanja

Prioritet:

```text
P1 / High
```

---

## 6. Tehnički dug: Security

### 6.1 Nije potvrđen permission model

Preporučeni model:

```text
Korisnici imaju read-only pristup SharePoint-u, a upise izvršavaju Power Automate flow-ovi preko service account-a.
```

Nepoznato:

- da li je ovaj model već primenjen
- koji service account se koristi
- ko ima edit prava nad listama
- ko ima pristup AppConfig listi
- ko može ručno menjati `RezervisaniBrojevi`
- ko može brisati dokumente

Rizik:

```text
Ako korisnici mogu direktno menjati SharePoint podatke, mogu zaobići poslovnu logiku iz Power Automate flow-ova.
```

Preporuka:

- uvesti read-only za standardne korisnike
- ograničiti edit prava na service account
- zaštititi `AppConfig`
- zaštititi `RezervisaniBrojevi`
- uvesti posebne admin role

Prioritet:

```text
P1 / High
```

---

### 6.2 Nije potvrđen audit model

Nepoznato:

- da li postoji audit lista
- da li se loguju flow run ID-jevi
- da li se loguje stvarni korisnik
- da li se loguje request/response
- da li se loguju greške

Rizik:

```text
Bez audit loga nije moguće pouzdano rekonstruisati ko je, kada i zašto dobio određeni delovodni broj.
```

Preporuka:

- kreirati `FlowExecutionLog`
- dodati `CorrelationId`
- logovati sve kritične akcije
- logovati uspešne i neuspešne rezervacije broja

Prioritet:

```text
P1 / High
```

---

## 7. Tehnički dug: Operativni rad

### 7.1 Nejasan Dev/Test/Prod proces

Nepoznato:

- da li postoje odvojena okruženja
- da li postoje različiti SharePoint site-ovi
- da li postoji ALM proces
- da li se koristi managed solution
- da li se koriste environment variables
- da li se connection references pravilno mapiraju

Rizik:

```text
Bez ALM procesa, promene u produkciji mogu biti ručne, teško proverljive i rizične.
```

Preporuka:

- definisati Dev/Test/Prod
- koristiti solution packaging
- dokumentovati connection references
- koristiti environment variables
- uvesti release checklist

Prioritet:

```text
P2 / Medium
```

---

### 7.2 Nije poznat monitoring

Nepoznato:

- da li se prati neuspeh flow-ova
- da li postoje email notifikacije za greške
- da li postoji dashboard grešaka
- da li postoji log za retry
- da li postoji vlasnik procesa

Preporuka:

- definisati monitoring flow-ova
- dodati dnevni pregled neuspešnih procesa
- dodati admin ekran `scrLogs`
- dodati alert za kritične greške u dodeli broja

Prioritet:

```text
P2 / Medium
```

---

## 8. Prioritetni backlog

### P0 / Critical

| ID | Stavka | Razlog |
|---|---|---|
| P0-001 | Concurrency-safe dodela delovodnog broja | Sprečavanje duplih brojeva |
| P0-002 | Unique key u `RezervisaniBrojevi` | Tehnička zaštita od duplikata |
| P0-003 | Unique / kontrola `DelovodniBroj` u `Shared Documents` | Finalna zaštita poslovnog ključa |
| P0-004 | Analiza flow-a `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` | Potvrda algoritma |
| P0-005 | Standardizovan error response za zavođenje | Pouzdan UX i dijagnostika |

---

### P1 / High

| ID | Stavka | Razlog |
|---|---|---|
| P1-001 | Audit log za sve kritične akcije | Praćenje i kontrola |
| P1-002 | Analiza `Upload Doc` flow-a | Stabilnost upload procesa |
| P1-003 | Analiza email intake procesa | Sprečavanje duplikata iz emaila |
| P1-004 | JSON schema za `AppConfig` | Stabilnost konfiguracije |
| P1-005 | Security model sa read-only korisnicima | Kontrola direktnih izmena |
| P1-006 | Dokumentovanje Canvas app ekrana | Razumevanje aplikacije |

---

### P2 / Medium

| ID | Stavka | Razlog |
|---|---|---|
| P2-001 | ALM dokumentacija | Stabilan release proces |
| P2-002 | Dev/Test/Prod razdvajanje | Kontrola promena |
| P2-003 | Monitoring flow-ova | Operativna stabilnost |
| P2-004 | Export verzionisanje | Kontrola konfiguracija |
| P2-005 | Admin ekran za logove | Lakša podrška |

---

## 9. Zaključak

Najveći tehnički dug i najveći poslovni rizik DocCentral rešenja odnosi se na dodelu delovodnog broja u uslovima istovremenog rada više korisnika.

Zato nova verzija mora prvo rešiti:

```text
Concurrency-safe numbering
```

Tek nakon toga treba širiti funkcionalnosti, UI i dodatne procese.

Prioritet razvoja:

1. sigurna dodela delovodnog broja
2. audit i logging
3. stabilan upload
4. sigurnosni model
5. email intake stabilizacija
6. AppConfig validacija
7. ALM i monitoring
