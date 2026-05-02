# 04 — AppConfig analiza

## 1. Svrha dokumenta

Ovaj dokument opisuje identifikovanu SharePoint listu **AppConfig** u okviru rešenja **DocCentral v6.0**.

Cilj dokumenta je da definiše:

- šta je potvrđeno iz SharePoint metadata odgovora
- koja je pretpostavljena uloga `AppConfig` liste
- koji su rizici korišćenja konfiguracije u tekstualnim JSON kolonama
- kako treba dokumentovati konfiguraciju u GitHub repozitorijumu
- kako treba unaprediti `AppConfig` model za enterprise verziju

---

## 2. Status analize

Status ovog dokumenta:

```text
DELIMIČNO POTVRĐENO
```

Razlog:

- SharePoint metadata potvrđuje postojanje liste `AppConfig`.
- SharePoint metadata potvrđuje postojanje kolona `Title`, `Config` i `ColumnHeader`.
- SharePoint metadata ne sadrži konkretne zapise iz liste.
- Sadržaj `Config` i `ColumnHeader` kolona nije dostavljen.
- Zato se poslovna uloga liste može delimično zaključiti, ali ne može biti potpuno potvrđena.

---

## 3. Identifikovana lista

Naziv liste:

```text
AppConfig
```

Scope:

```text
/sites/DocumentCentralv6.0/Lists/AppConfig
```

REST context:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 4. Potvrđena polja

| Internal name | Display name | Tip | Read-only | Required | Status |
|---|---|---|---|---|---|
| `Title` | Title | Single line of text | Ne | Ne | Potvrđeno |
| `Config` | Config | Multiple lines of text | Ne | Ne | Potvrđeno |
| `ColumnHeader` | ColumnHeader | Multiple lines of text | Ne | Ne | Potvrđeno |
| `ID` | ID | Counter | Da | Ne | Potvrđeno |
| `Created` | Created | Date and Time | Da | Ne | Potvrđeno |
| `Modified` | Modified | Date and Time | Da | Ne | Potvrđeno |
| `Author` | Created By | Person or Group | Da | Ne | Potvrđeno |
| `Editor` | Modified By | Person or Group | Da | Ne | Potvrđeno |
| `Attachments` | Attachments | Attachments | Ne | Ne | Potvrđeno |
| `ContentType` | Content Type | Computed | Ne | Ne | Potvrđeno |

---

## 5. Tehnički detalji potvrđeni za `Config`

Polje:

```text
InternalName: Config
DisplayName: Config
TypeAsString: Note
TypeDisplayName: Multiple lines of text
FieldTypeKind: 3
RichText: false
AppendOnly: false
Filterable: false
Sortable: false
```

Zaključak:

`Config` je obična multiple lines of text kolona bez rich text formata.

Ovo je pogodno za čuvanje JSON teksta, ali SharePoint ne validira da li je sadržaj zaista ispravan JSON.

---

## 6. Tehnički detalji potvrđeni za `ColumnHeader`

Polje:

```text
InternalName: ColumnHeader
DisplayName: ColumnHeader
TypeAsString: Note
TypeDisplayName: Multiple lines of text
FieldTypeKind: 3
RichText: false
AppendOnly: false
Filterable: false
Sortable: false
```

Zaključak:

`ColumnHeader` je takođe obična multiple lines of text kolona.

Pretpostavka je da se koristi za konfiguraciju prikaza kolona, labela ili tabela.

---

## 7. Pretpostavljena poslovna uloga liste

Na osnovu naziva `AppConfig` i potvrđenih polja, lista verovatno služi za centralno čuvanje konfiguracije aplikacije.

Mogući tipovi konfiguracije:

- opšta podešavanja aplikacije
- konfiguracija delovodnih knjiga
- konfiguracija tipova dokumenata
- konfiguracija statusa
- konfiguracija UI kolona
- konfiguracija prevoda
- konfiguracija korisnika i rola
- konfiguracija organizacionih jedinica
- konfiguracija pravila za zavođenje
- konfiguracija Power Apps kolekcija
- konfiguracija Power Automate flow ponašanja

Status: **pretpostavka**

Razlog: XML metadata ne sadrži konkretne iteme iz liste.

---

## 8. Verovatni obrazac korišćenja u Power Apps aplikaciji

U Canvas aplikacijama ovakav model se najčešće koristi ovako:

1. Aplikacija pri startu učita sve ili izabrane zapise iz `AppConfig`.
2. Power Apps pročita `Config` tekstualno polje.
3. JSON se parsira pomoću `ParseJSON`.
4. Na osnovu parsed vrednosti kreiraju se lokalne kolekcije.
5. Aplikacija koristi kolekcije za:
   - dropdown vrednosti
   - prevode
   - UI podešavanja
   - poslovna pravila
   - inicijalna stanja
   - filtere
   - permission logiku

Primer konceptualnog Power Apps obrasca:

```powerfx
ClearCollect(
    colAppConfig,
    AppConfig
);

Set(
    gblConfig,
    ParseJSON(
        LookUp(
            colAppConfig,
            Title = "Settings"
        ).Config
    )
);
```

Status: **pretpostavka**

---

## 9. Prednosti trenutnog pristupa

Korišćenje `AppConfig` liste ima nekoliko prednosti:

| Prednost | Objašnjenje |
|---|---|
| Centralna konfiguracija | Podešavanja nisu hardkodovana u aplikaciji |
| Brža izmena | Neke vrednosti se mogu promeniti bez publish-a aplikacije |
| Fleksibilnost | Jedna lista može sadržati različite konfiguracije |
| Jednostavan storage | SharePoint lista je dostupna bez premium konektora |
| Pogodno za prevode | JSON može sadržati višejezične vrednosti |

---

## 10. Glavni rizici trenutnog pristupa

### 10.1 Nevalidan JSON

Ako korisnik ručno izmeni `Config` i napravi sintaksnu grešku, aplikacija može prestati da radi.

Primer greške:

```json
{
  "Language": "sr-Latn-RS",
  "Theme": "Default",
}
```

Problem:

Zarez posle poslednjeg property-ja čini JSON nevalidnim.

---

### 10.2 Ne postoji potvrđena JSON šema

Bez šeme nije jasno:

- koja polja su obavezna
- koji tip podataka se očekuje
- koje vrednosti su dozvoljene
- šta je breaking change
- koja verzija aplikacije koristi koju konfiguraciju

---

### 10.3 Prevelik broj različitih konfiguracija u jednom polju

Ako se u jedan `Config` zapis stavi previše nepovezanih podešavanja, održavanje postaje teško.

Loš primer:

```json
{
  "translations": [],
  "users": [],
  "roles": [],
  "documentTypes": [],
  "numbering": [],
  "theme": {},
  "flowEndpoints": {},
  "permissions": []
}
```

Bolji pristup je razdvajanje po tipu konfiguracije.

---

### 10.4 Nema verzionisanja konfiguracije

Ako aplikacija očekuje novu strukturu, a u `AppConfig` je stara struktura, može doći do greške.

Primer:

Aplikacija očekuje:

```json
{
  "schemaVersion": "2.0",
  "items": []
}
```

A u listi postoji:

```json
[
  {
    "Title": "Old format"
  }
]
```

---

### 10.5 Nema kontrolisanog deployment procesa

Ako se konfiguracija menja direktno u SharePoint listi, nema jasnog procesa:

- ko je odobrio izmenu
- šta je promenjeno
- da li je testirano
- kako se vraća prethodna verzija
- da li je izmena povezana sa GitHub commit-om

---

## 11. Preporučeni enterprise model za `AppConfig`

Za enterprise verziju preporučuje se proširenje liste `AppConfig`.

### 11.1 Predložena dodatna polja

| Internal name | Tip | Obavezno | Namena |
|---|---|---|---|
| `ConfigKey` | Single line of text | Da | Jedinstveni ključ konfiguracije |
| `ConfigType` | Choice / Single line of text | Da | Tip konfiguracije |
| `Config` | Multiple lines of text | Da | JSON konfiguracija |
| `SchemaVersion` | Single line of text | Da | Verzija JSON šeme |
| `AppVersion` | Single line of text | Ne | Verzija aplikacije |
| `Environment` | Choice | Da | Dev/Test/Prod |
| `IsActive` | Yes/No | Da | Da li se koristi |
| `Description` | Multiple lines of text | Ne | Opis namene |
| `ValidationStatus` | Choice | Da | Valid/Invalid/Not checked |
| `LastValidatedOn` | Date and Time | Ne | Kada je validirano |
| `LastValidatedByFlowRunId` | Single line of text | Ne | Flow koji je validirao |
| `ChangeTicket` | Single line of text | Ne | Interni zahtev/tiket |
| `ApprovedBy` | Person or Group | Ne | Ko je odobrio |
| `ApprovedOn` | Date and Time | Ne | Kada je odobreno |

---

## 12. Predloženi `ConfigType` tipovi

Predloženi tipovi konfiguracije:

| ConfigType | Namena |
|---|---|
| `Settings` | Globalna podešavanja aplikacije |
| `Translations` | Prevodi i lokalizacija |
| `DocumentTypes` | Tipovi dokumenata |
| `NumberingBooks` | Delovodne knjige |
| `Statuses` | Statusi dokumenata |
| `Permissions` | Role i dozvole |
| `Users` | Konfiguracija korisnika |
| `OrganizationalUnits` | Organizacione jedinice |
| `ColumnHeaders` | Konfiguracija kolona |
| `Theme` | UI boje i tema |
| `Flows` | Flow identifikatori ili endpoint podešavanja |
| `ValidationRules` | Pravila validacije |
| `FeatureFlags` | Uključivanje/isključivanje funkcionalnosti |

---

## 13. Preporučena JSON struktura

Svaki `Config` zapis treba da ima standardizovan wrapper.

Primer:

```json
{
  "schemaVersion": "1.0",
  "configType": "Settings",
  "environment": "prod",
  "lastUpdated": "2026-04-30",
  "items": []
}
```

Za jednostavna podešavanja:

```json
{
  "schemaVersion": "1.0",
  "configType": "Settings",
  "environment": "prod",
  "settings": {
    "defaultLanguage": "sr-Latn-RS",
    "supportedLanguages": [
      "sr-Latn-RS",
      "en-US"
    ],
    "defaultPageSize": 50,
    "enableEmailIntake": true,
    "enableExports": true
  }
}
```

Za delovodne knjige:

```json
{
  "schemaVersion": "1.0",
  "configType": "NumberingBooks",
  "environment": "prod",
  "items": [
    {
      "key": "OPSTA",
      "title": "Opšta delovodna knjiga",
      "prefix": "OP",
      "year": 2026,
      "isActive": true
    }
  ]
}
```

Za prevode:

```json
{
  "schemaVersion": "1.0",
  "configType": "Translations",
  "environment": "prod",
  "items": [
    {
      "key": "document.register",
      "sr-Latn-RS": "Zavedite dokument",
      "en-US": "Register document"
    }
  ]
}
```

---

## 14. Preporučeni GitHub folder za konfiguracije

U GitHub repozitorijumu preporučuje se folder:

```text
config/
```

Predložena struktura:

```text
config/
├── appconfig-index.md
├── schemas/
│   ├── settings.schema.json
│   ├── translations.schema.json
│   ├── numbering-books.schema.json
│   ├── document-types.schema.json
│   ├── permissions.schema.json
│   └── column-headers.schema.json
│
├── examples/
│   ├── settings.example.json
│   ├── translations.example.json
│   ├── numbering-books.example.json
│   ├── document-types.example.json
│   ├── permissions.example.json
│   └── column-headers.example.json
│
└── production-snapshots/
    └── README.md
```

Napomena:

Ne preporučuje se čuvanje poverljivih produkcionih vrednosti u public repozitorijumu.

---

## 15. Pravilo za poverljive vrednosti

`AppConfig` ne bi trebalo da sadrži:

- lozinke
- client secret vrednosti
- refresh tokene
- API ključeve
- connection string-ove
- privatne URL-ove koji otkrivaju sigurnosnu arhitekturu
- lične podatke koji nisu neophodni
- podatke koji spadaju u poverljive poslovne informacije

Ako postoje poverljive vrednosti, treba ih izmestiti u:

- environment variables u Power Platform solution-u
- Azure Key Vault
- sigurnu konfiguraciju flow-a
- tenant-level sigurnosni mehanizam

---

## 16. AppConfig i environment variables

Za novu enterprise verziju treba jasno razdvojiti:

### 16.1 Environment variables

Koristiti za:

- site URL
- nazive lista
- nazive biblioteka
- environment-specific vrednosti
- flow endpoint/logical name vrednosti
- konfiguraciju koja se razlikuje između Dev/Test/Prod

### 16.2 AppConfig

Koristiti za:

- poslovna pravila
- prevode
- UI podešavanja
- tipove dokumenata
- statuse
- role
- parametre koje poslovni admin može da menja

### 16.3 Ne koristiti AppConfig za

- sigurnosne tajne
- tehničke connection reference
- vrednosti koje moraju biti deo deployment pipeline-a
- vrednosti koje ne sme menjati poslovni korisnik

---

## 17. AppConfig i Power Apps

Power Apps treba da učitava `AppConfig` kontrolisano.

Preporučeni obrazac:

1. Učitati samo aktivne konfiguracije.
2. Validirati da `Config` nije prazan.
3. Parsirati JSON.
4. Ako JSON nije validan, prikazati kontrolisanu grešku.
5. Upisati grešku u audit.
6. Ne dozvoliti nastavak rada ako je kritična konfiguracija nevalidna.

Konceptualni Power Fx obrazac:

```powerfx
ClearCollect(
    colAppConfigRaw,
    Filter(
        AppConfig,
        IsActive = true
    )
);
```

Napomena:

Ovo je ciljna preporuka. Trenutna lista nema potvrđeno polje `IsActive`.

---

## 18. AppConfig i Power Automate

Power Automate flow-ovi mogu koristiti `AppConfig` za:

- čitanje poslovnih parametara
- definisanje statusnih vrednosti
- čitanje konfiguracije delovodnih knjiga
- čitanje pravila validacije
- čitanje email intake podešavanja
- čitanje export parametara

Međutim, za kritične operacije kao što je generisanje delovodnog broja, sama konfiguracija iz `AppConfig` nije dovoljna.

Za delovodne brojeve mora postojati poseban concurrency-safe mehanizam.

---

## 19. AppConfig i delovodni brojevi

Ako `AppConfig` sadrži poslednji iskorišćeni broj ili counter, to je visokorizično.

Nije preporučeno:

```text
Učitati JSON iz AppConfig -> pročitati poslednji broj -> povećati za 1 -> upisati JSON nazad
```

Razlog:

Kod paralelnog rada više korisnika može doći do race condition problema.

Preporučeno:

- `AppConfig` sme da čuva pravila formatiranja broja
- `AppConfig` sme da čuva listu delovodnih knjiga
- `AppConfig` sme da čuva prefikse
- `AppConfig` ne treba da bude jedini izvor za atomic counter state

Za counter state treba koristiti posebno dizajniran servis/listu, na primer:

```text
NumberingCounters
```

---

## 20. Predložena podela odgovornosti

| Element | Odgovornost |
|---|---|
| `AppConfig` | Poslovna konfiguracija |
| `NumberingCounters` | Trenutno stanje brojača |
| `RezervisaniBrojevi` | Rezervacije i istorija brojeva |
| `Shared Documents` | Finalni dokumenti |
| `AuditLog` | Događaji, greške i trace |
| Power Apps | UI i pozivanje backend logike |
| Power Automate | Validacija, upis, numbering, audit |

---

## 21. Minimalna validaciona pravila

Za svaki `AppConfig` zapis treba validirati:

| Pravilo | Objašnjenje |
|---|---|
| `Title` nije prazan | Svaka konfiguracija mora imati naziv |
| `Config` nije prazan | Ne sme biti prazan JSON |
| JSON je validan | Mora proći parsing |
| `schemaVersion` postoji | Mora biti jasno koja šema važi |
| `configType` postoji | Mora biti jasno šta konfiguracija predstavlja |
| `environment` postoji | Mora biti jasno za koje okruženje važi |
| očekivana polja postoje | Prema tipu konfiguracije |
| nema duplih key vrednosti | Posebno za prevode, statuse, tipove dokumenata |
| nema nedozvoljenih vrednosti | Na primer nepoznat jezik ili status |
| nema poverljivih vrednosti | Ne smeju se držati tajne |

---

## 22. Predlog validacionog flow-a

Predloženi flow:

```text
Validate AppConfig Item
```

Trigger:

```text
When an item is created or modified
```

Logika:

1. Proveri da li je `Config` prazan.
2. Pokušaj parsiranje JSON-a.
3. Proveri `schemaVersion`.
4. Proveri `configType`.
5. Učitaj odgovarajuću JSON šemu.
6. Validiraj obavezna polja.
7. Ako je validno:
   - postavi `ValidationStatus = Valid`
   - postavi `LastValidatedOn`
8. Ako nije validno:
   - postavi `ValidationStatus = Invalid`
   - upiši grešku u `ErrorMessage`
   - upiši događaj u `AuditLog`

Napomena:

Trenutna lista nema potvrđena polja `ValidationStatus`, `LastValidatedOn` i `ErrorMessage`. To su preporuke za novu verziju.

---

## 23. Predlog dokumentovanja AppConfig zapisa

Za svaki konfiguracioni zapis treba napraviti dokumentaciju u formatu:

```text
config/appconfig-index.md
```

Primer tabele:

| ConfigKey | ConfigType | Opis | Schema | Primer |
|---|---|---|---|---|
| `settings.global` | Settings | Globalna podešavanja | `settings.schema.json` | `settings.example.json` |
| `translations.main` | Translations | Prevodi aplikacije | `translations.schema.json` | `translations.example.json` |
| `numbering.books` | NumberingBooks | Delovodne knjige | `numbering-books.schema.json` | `numbering-books.example.json` |

---

## 24. Predlog dokumentovanja JSON šeme

Primer JSON schema fajla:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "DocCentral Settings Config",
  "type": "object",
  "required": [
    "schemaVersion",
    "configType",
    "environment",
    "settings"
  ],
  "properties": {
    "schemaVersion": {
      "type": "string"
    },
    "configType": {
      "type": "string",
      "const": "Settings"
    },
    "environment": {
      "type": "string",
      "enum": [
        "dev",
        "test",
        "prod"
      ]
    },
    "settings": {
      "type": "object"
    }
  }
}
```

---

## 25. Deployment preporuke

Za enterprise verziju preporučuje se sledeći proces:

1. Izmena konfiguracije se prvo priprema u GitHub-u.
2. JSON se validira lokalno ili kroz CI.
3. Pull request pregleda tehničko lice.
4. Poslovni vlasnik odobrava izmenu.
5. Konfiguracija se deploy-uje u test okruženje.
6. Radi se test aplikacije i flow-ova.
7. Nakon potvrde, konfiguracija se deploy-uje u produkciju.
8. Produkcioni `AppConfig` zapis dobija novu verziju.
9. Audit log beleži deployment.

---

## 26. AppConfig rollback

Rollback treba da bude moguć.

Preporučeni model:

- svaka promena ima verziju
- prethodna verzija ostaje dostupna
- samo jedna verzija istog `ConfigKey` može biti aktivna po environment-u
- rollback se radi promenom aktivne verzije ili ponovnim deployment-om prethodne konfiguracije

Primer:

| ConfigKey | Version | Environment | IsActive |
|---|---:|---|---|
| `settings.global` | 1.0 | prod | false |
| `settings.global` | 1.1 | prod | true |

---

## 27. Bezbednosne preporuke

Preporuke:

- ograničiti ko može da menja `AppConfig`
- obični korisnici ne treba da imaju edit prava
- izmene treba da idu kroz kontrolisan proces
- uključiti version history na listi
- uključiti audit
- razmotriti approval za kritične izmene
- validirati JSON pre nego što ga aplikacija koristi
- ne čuvati tajne u `Config` polju

---

## 28. Performance preporuke

Rizici:

- učitavanje cele `AppConfig` liste pri svakom startu aplikacije
- parsiranje velikih JSON struktura u Power Apps-u
- nepostojanje filterable/sortable mogućnosti nad `Config`
- preveliki JSON payload

Preporuke:

- razdvojiti konfiguracije po tipu
- učitavati samo potrebne konfiguracije
- koristiti `Title` ili `ConfigKey` za lookup
- držati JSON dovoljno malim
- ne čuvati velike master data setove u jednom `Config` polju
- za velike referentne podatke koristiti posebne SharePoint liste ili Dataverse

---

## 29. Kritična pravila za novu verziju

1. `AppConfig` ne sme biti jedini izvor istine za generisanje delovodnog broja.
2. `AppConfig` mora imati jasnu šemu.
3. Svaki JSON mora biti validiran.
4. Svaka konfiguracija mora imati verziju.
5. Produkcione izmene moraju biti auditovane.
6. Obični korisnici ne smeju ručno menjati kritične konfiguracije.
7. Poverljive vrednosti ne smeju biti u `AppConfig`.
8. Konfiguracije koje utiču na upis dokumenata moraju imati rollback plan.

---

## 30. Zaključak

`AppConfig` je važan deo trenutne arhitekture DocCentral v6.0 rešenja.

Potvrđeno je da lista postoji i da ima kolone `Config` i `ColumnHeader`, koje su tehnički pogodne za čuvanje JSON ili tekstualne konfiguracije.

Međutim, bez šeme, verzionisanja, validacije i kontrolisanog deployment procesa, `AppConfig` može postati značajan izvor tehničkog duga.

Za novu enterprise verziju potrebno je zadržati koncept centralne konfiguracije, ali ga formalizovati kroz:

- tipove konfiguracije
- JSON šeme
- verzionisanje
- validaciju
- audit
- GitHub dokumentaciju
- kontrolisan deployment
- jasnu podelu između AppConfig, environment variables i sigurnih tajni

---

## 31. Otvorena pitanja

1. Koji konkretni zapisi postoje u `AppConfig` listi?
2. Da li je `Config` uvek JSON?
3. Da li je `ColumnHeader` JSON ili slobodan tekst?
4. Da li Power Apps direktno parsira `Config`?
5. Da li Power Automate flow-ovi čitaju `AppConfig`?
6. Da li `AppConfig` sadrži delovodne knjige?
7. Da li `AppConfig` sadrži poslednji broj brojača?
8. Da li `AppConfig` sadrži prevode?
9. Da li postoji razlika između Dev/Test/Prod konfiguracije?
10. Ko ima pravo izmene `AppConfig` liste?
11. Da li je uključena version history na listi?
12. Da li postoji proces odobravanja izmene konfiguracije?
13. Da li postoji backup konfiguracije?
14. Da li su konfiguracije dokumentovane van SharePoint-a?
15. Da li postoje tajne ili osetljive vrednosti u `Config` polju?

---

## 32. Sledeći dokument

Sledeći dokument:

```text
05-document-libraries.md
```

Taj dokument treba detaljno da opiše biblioteke:

- `Shared Documents`
- `EmailDocuments`
- `Exports`

i njihovu ulogu u dokumentnom lifecycle-u.
