# DocCentral v6.0

## Tehnička dokumentacija i razvojna osnova

Ovaj repozitorijum sadrži tehničku dokumentaciju za Power Platform rešenje **DocCentral v6.0**.

Dokumentacija je pripremljena kao osnova za:

- razumevanje postojeće aplikacije,
- dokumentovanje Power Apps, Power Automate i SharePoint data modela,
- identifikovanje tehničkog duga i rizika,
- razvoj nove enterprise verzije aplikacije,
- pripremu razvojnih instrukcija za Cloud Code / Claude Code / AI-assisted development.

---

## 1. Svrha dokumentacije

Cilj dokumentacije je da omogući jasno razumevanje trenutnog stanja rešenja i da posluži kao tehnička osnova za dalji razvoj.

Dokumentacija pokriva:

1. Business overview.
2. Trenutnu arhitekturu rešenja.
3. SharePoint data model.
4. Ključne liste i biblioteke.
5. Logiku delovodnog broja.
6. Rizike i tehnički dug.
7. Enterprise preporuke.
8. Osnovu za Cloud Code razvoj nove verzije.

---

## 2. Opis rešenja

**DocCentral v6.0** je Power Platform rešenje za elektronsku pisarnicu i centralnu evidenciju dokumenata.

Na osnovu dostavljenih SharePoint REST XML metadata odgovora, rešenje koristi **SharePoint Online** kao primarni data layer i storage layer.

Identifikovane komponente:

- SharePoint liste
- SharePoint document libraries
- Power Apps Canvas aplikacija
- Power Automate flow-ovi
- konfiguraciona lista `AppConfig`
- lista za rezervaciju brojeva `RezervisaniBrojevi`
- glavna biblioteka dokumenata
- email document biblioteka
- export biblioteka

---

## 3. SharePoint lokacija

### Site URL

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

### REST API base URL

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 4. Identifikovane SharePoint liste i biblioteke

Na osnovu dostavljenih XML metadata odgovora identifikovane su sledeće liste i biblioteke:

| Naziv | Tip | Scope | Namena |
|---|---|---|---|
| `RezervisaniBrojevi` | SharePoint lista | `/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi` | Rezervacija / kontrola delovodnih brojeva |
| `AppConfig` | SharePoint lista | `/sites/DocumentCentralv6.0/Lists/AppConfig` | Centralna konfiguracija aplikacije |
| `Shared Documents` | Document library | `/sites/DocumentCentralv6.0/Shared Documents` | Glavna biblioteka dokumenata |
| `EmailDocuments` | Document library | `/sites/DocumentCentralv6.0/EmailDocuments` | Dokumenti nastali iz email procesa |
| `Exports` | Document library | `/sites/DocumentCentralv6.0/Exports` | Exportovani fajlovi / izveštaji |

---

## 5. Ključni poslovni zahtev

Najvažniji enterprise zahtev:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovaj zahtev ima najviši prioritet u novoj verziji rešenja.

To znači:

- delovodni broj se ne sme generisati samo u Canvas aplikaciji,
- delovodni broj se ne sme generisati lokalno u kolekciji,
- delovodni broj se ne sme zasnivati samo na poslednjem pročitanom broju iz SharePoint liste bez kontrole konkurentnosti,
- mora postojati centralizovan servis za generisanje brojeva,
- mora postojati atomic / optimistic locking mehanizam,
- mora postojati retry logika,
- mora postojati audit log,
- mora postojati unique constraint na finalnom delovodnom broju.

---

## 6. Visok nivo arhitekture

Trenutna arhitektura se može opisati ovako:

```text
Power Apps Canvas App
        |
        | poziva / čita / šalje podatke
        v
Power Automate flows
        |
        | kreiraju, ažuriraju, validiraju
        v
SharePoint Online
        |
        | liste + biblioteke
        v
Dokumenti, konfiguracija, delovodni brojevi, exporti
```

---

## 7. Glavni SharePoint elementi

### 7.1 `AppConfig`

Lista `AppConfig` sadrži centralnu konfiguraciju aplikacije.

#### Identifikovana polja

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

#### Zaključak

`Config` polje najverovatnije sadrži JSON konfiguracije koje aplikacija koristi za inicijalizaciju kolekcija, podešavanja, prevode i poslovna pravila.

#### Status

**POTVRĐENO DELIMIČNO**

XML potvrđuje postojanje kolone `Config`, ali konkretan sadržaj JSON konfiguracije nije dostavljen.

---

### 7.2 `RezervisaniBrojevi`

Lista `RezervisaniBrojevi` je posebno važna za enterprise zahtev vezan za jedinstvene delovodne brojeve.

#### Identifikovana polja

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

#### Zaključak

Lista najverovatnije služi za rezervaciju brojeva pre finalnog kreiranja dokumenta ili predmeta.

#### Status

**PRETPOSTAVKA**

Naziv liste i polja ukazuju na rezervaciju brojeva, ali nije dostavljena logika flow-a koji koristi ovu listu.

---

### 7.3 `Shared Documents`

Biblioteka `Shared Documents` je glavna biblioteka dokumenata.

#### Identifikovana polja

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

#### Zaključak

Ova biblioteka čuva dokumente koji se zavode u sistemu i povezuje ih sa delovodnim brojem.

---

### 7.4 `EmailDocuments`

Biblioteka `EmailDocuments` je posebna biblioteka za dokumente dobijene ili kreirane iz email procesa.

#### Identifikovana polja

| Internal name | Display name | Tip |
|---|---|---|
| `FileLeafRef` | Name | File |
| `Title` | Title | Single line of text |
| `_ExtendedDescription` | Description | Multiple lines of text |
| `PosiljalacEmail` | PosiljalacEmail | Single line of text |
| `Posiljalac` | Posiljalac | Single line of text |
| `MediaServiceImageTags` | Image Tags | Managed Metadata |
| `ContentType` | Content Type | Computed |

#### Zaključak

Ova biblioteka verovatno služi kao ulazni kanal za dokumente pristigle emailom.

#### Status

**PRETPOSTAVKA**

Naziv biblioteke i polja `PosiljalacEmail` i `Posiljalac` ukazuju na email intake proces, ali nije dostavljen flow koji potvrđuje kompletnu logiku.

---

### 7.5 `Exports`

Biblioteka `Exports` služi za generisane fajlove i exporte.

#### Identifikovana polja

| Internal name | Display name | Tip |
|---|---|---|
| `FileLeafRef` | Name | File |
| `Title` | Title | Single line of text |
| `_ExtendedDescription` | Description | Multiple lines of text |
| `ContentType` | Content Type | Computed |

#### Zaključak

Biblioteka je verovatno namenjena za generisane izveštaje, exporte ili dokumentaciju koju sistem proizvodi.

#### Status

**PRETPOSTAVKA**

---

## 8. Ključni rizici

### 8.1 Rizik duplog delovodnog broja

Najveći rizik sistema je situacija u kojoj dva korisnika istovremeno zavode dokument i dobiju isti delovodni broj.

Ovo mora biti rešeno na nivou backend logike, a ne samo na nivou Power Apps forme.

#### Obavezna zaštita

```text
Atomic counter update
Optimistic concurrency
Retry policy
Unique constraint
Audit log
Controlled error response
```

---

### 8.2 Rizik od logike u Canvas aplikaciji

Ako Canvas aplikacija sama računa sledeći delovodni broj, postoji rizik od race condition problema.

Canvas aplikacija sme da:

- prikaže formu,
- validira obavezna polja,
- pozove flow,
- prikaže rezultat.

Canvas aplikacija ne sme samostalno da:

- određuje konačni delovodni broj,
- garantuje jedinstvenost broja,
- vrši finalnu rezervaciju bez server-side potvrde.

---

### 8.3 Rizik od SharePoint ograničenja

SharePoint Online može biti dobar data layer za manja i srednja rešenja, ali kod enterprise pisarnice treba obratiti pažnju na:

- delegaciju,
- list view threshold,
- indexing,
- unique constraints,
- item-level permissions,
- audit,
- race conditions,
- retry i throttling,
- flow timeout,
- skalabilnost.

---

## 9. Predlog strukture GitHub repozitorijuma

```text
doccentral-v6-documentation/
│
├── README.md
│
├── docs/
│   ├── 01-business-overview.md
│   ├── 02-current-architecture.md
│   ├── 03-sharepoint-data-model.md
│   ├── 04-appconfig-analysis.md
│   ├── 05-document-libraries.md
│   ├── 06-numbering-and-concurrency.md
│   ├── 07-power-apps-analysis.md
│   ├── 08-power-automate-analysis.md
│   ├── 09-security-permissions.md
│   ├── 10-known-issues-technical-debt.md
│   ├── 11-enterprise-improvements.md
│   └── 12-cloud-code-development-brief.md
│
├── architecture/
│   ├── current-architecture.md
│   ├── target-architecture.md
│   └── numbering-sequence.md
│
├── data-model/
│   ├── appconfig.md
│   ├── rezervisani-brojevi.md
│   ├── shared-documents.md
│   ├── email-documents.md
│   └── exports.md
│
├── prompts/
│   └── cloud-code-master-prompt.md
│
└── backlog/
    ├── known-issues.md
    ├── enterprise-backlog.md
    └── open-questions.md
```

---

## 10. Pravila dokumentacije

Svi dokumenti u ovom repozitorijumu moraju jasno razdvajati:

- činjenice,
- pretpostavke,
- nepoznato,
- rizike,
- preporuke.

Ne sme se izmišljati logika koja nije potvrđena iz:

- solution fajla,
- XML metadata odgovora,
- Power Apps formula,
- Power Automate definicija,
- korisničkog objašnjenja.

Ako nešto nije jasno, mora biti označeno kao:

```text
NEPOZNATO
```

---

## 11. Trenutno poznato

### Činjenice

- Rešenje koristi SharePoint Online site `DocumentCentralv6.0`.
- Postoji lista `AppConfig`.
- Postoji lista `RezervisaniBrojevi`.
- Postoji biblioteka `Shared Documents`.
- Postoji biblioteka `EmailDocuments`.
- Postoji biblioteka `Exports`.
- `Shared Documents` ima kolonu `DelovodniBroj`.
- `Shared Documents` ima kolonu `Edokument`.
- `Shared Documents` ima kolonu `EdokumentID`.
- `EmailDocuments` ima kolone `Posiljalac` i `PosiljalacEmail`.
- `AppConfig` ima tekstualno višelinijsko polje `Config`.
- `RezervisaniBrojevi` ima numeričko polje `RezervisaniBroj`.

### Pretpostavke

- `AppConfig.Config` sadrži JSON konfiguraciju aplikacije.
- `RezervisaniBrojevi` se koristi za rezervaciju delovodnih brojeva.
- `EmailDocuments` se koristi za email intake proces.
- `Exports` se koristi za generisane export fajlove.

### Nepoznato

- Tačna Power Apps struktura ekrana.
- Tačne kolekcije u Canvas aplikaciji.
- Tačni Power Automate flow-ovi.
- Triggeri flow-ova.
- Koji flow generiše delovodni broj.
- Da li trenutno postoji ETag / optimistic concurrency.
- Da li postoji unique constraint nad `DelovodniBroj`.
- Da li postoji audit log.
- Da li postoji retry mehanizam.
- Da li korisnici direktno pišu u SharePoint ili isključivo preko flow-ova.

---

## 12. Prioriteti za dalju analizu

1. Analiza SharePoint data modela.
2. Analiza `AppConfig` konfiguracije.
3. Analiza `RezervisaniBrojevi` i logike delovodnog broja.
4. Analiza document libraries.
5. Analiza Canvas aplikacije.
6. Analiza Power Automate flow-ova.
7. Definisanje target enterprise arhitekture.
8. Pisanje Cloud Code master prompt-a.

---

## 13. Najvažniji zaključak

DocCentral v6.0 mora u novoj enterprise verziji imati centralizovan i concurrency-safe mehanizam za generisanje delovodnog broja.

To je najkritičniji deo sistema, jer se direktno odnosi na pravnu, poslovnu i operativnu ispravnost elektronske pisarnice.
