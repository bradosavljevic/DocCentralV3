# Data Model — Exports

## 1. Svrha dokumenta

Ovaj dokument opisuje SharePoint document library:

```text
Exports
```

U rešenju **DocCentral v6.0**, biblioteka `Exports` je namenjena za export fajlove, izveštaje i izvoz konfiguracionih podataka.

Korisnik je potvrdio:

```text
Exporti su export šifarnika i konfiguracije iz AppConfig-a.
```

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Postojanje biblioteke `Exports` | POTVRĐENO |
| SharePoint scope biblioteke | POTVRĐENO |
| Namena za export šifarnika | POTVRĐENO OD KORISNIKA |
| Namena za export konfiguracije iz `AppConfig` | POTVRĐENO OD KORISNIKA |
| Postojanje custom poslovnih metadata polja | NIJE POTVRĐENO |
| Tačan naziv flow-a za export | NEPOZNATO |
| Format export fajlova | NEPOZNATO |
| Da li export pokreće korisnik iz Canvas aplikacije | NEPOZNATO |
| Da li export radi automatski/scheduled | NEPOZNATO |

---

## 3. SharePoint lokacija

Site:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

Biblioteka:

```text
/sites/DocumentCentralv6.0/Exports
```

List GUID iz dostavljenog XML-a:

```text
8eda7e2f-5ece-4ec7-825b-dce232137b33
```

REST endpoint obrazac:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/Web/Lists(guid'8eda7e2f-5ece-4ec7-825b-dce232137b33')
```

---

## 4. Poslovna uloga biblioteke

`Exports` služi za čuvanje fajlova koji nastaju kao rezultat export procesa.

Potvrđena namena:

```text
Export šifarnika i konfiguracije iz AppConfig liste.
```

To znači da biblioteka najverovatnije sadrži:

- export konfiguracionih JSON vrednosti
- export šifarnika
- pomoćne fajlove za backup konfiguracije
- fajlove za migraciju konfiguracije između okruženja
- dokumentaciju ili snapshot stanja aplikacije

---

## 5. Identifikovana polja

Na osnovu dostavljenog SharePoint REST XML metadata odgovora, identifikovana su sledeća polja:

| Internal name | Display name | Tip | Required | Read only | Filterable | Napomena |
|---|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Da | Naziv export fajla |
| `Title` | Title | Single line of text | Ne | Ne | Da | SharePoint naslov |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Ne | Opis fajla |
| `ContentType` | Content Type | Computed | Ne | Ne | Da | SharePoint content type |

---

## 6. Zaključak o data modelu

Biblioteka `Exports` trenutno izgleda kao minimalna SharePoint document library bez potvrđenih custom poslovnih metadata polja za export proces.

Potvrđena custom ili poslovna polja nisu identifikovana u dostavljenom XML-u.

Zaključak:

```text
Exports biblioteka trenutno ima osnovna document library polja.
```

Mogući razlog:

```text
Export fajlovi se možda razlikuju po nazivu fajla, folderu ili sadržaju, a ne po SharePoint metadata kolonama.
```

---

## 7. Detalji ključnih polja

## 7.1 FileLeafRef

SharePoint metadata:

```text
InternalName: FileLeafRef
DisplayName: Name
TypeAsString: File
FieldTypeKind: 18
Required: true
ReadOnlyField: false
Filterable: true
```

Namena:

```text
Čuva naziv exportovanog fajla.
```

Primeri mogućih vrednosti:

```text
appconfig-export-2026-04-30.json
sifarnici-export-2026-04-30.xlsx
translations-export.json
document-types-export.csv
```

Preporuka:

```text
Naziv fajla treba da bude standardizovan i da sadrži tip exporta, datum i eventualno environment.
```

Primer standarda:

```text
{exportType}-{environment}-{yyyyMMdd-HHmmss}.{extension}
```

Primer:

```text
appconfig-prod-20260430-185913.json
```

---

## 7.2 Title

SharePoint metadata:

```text
InternalName: Title
DisplayName: Title
TypeAsString: Text
FieldTypeKind: 2
Required: false
ReadOnlyField: false
Filterable: true
Sealed: true
MaxLength: 255
```

XML pokazuje:

```text
ShowInNewForm="FALSE"
ShowInFileDlg="FALSE"
Sealed="TRUE"
```

Zaključak:

```text
Title nije primarno polje za export logiku.
```

Preporuka:

```text
Ne oslanjati se na Title za klasifikaciju export fajlova.
```

---

## 7.3 _ExtendedDescription

SharePoint metadata:

```text
InternalName: _ExtendedDescription
DisplayName: Description
TypeAsString: Note
FieldTypeKind: 3
RichText: true
Required: false
ReadOnlyField: false
Filterable: false
```

Namena:

```text
Može služiti za opis export fajla.
```

Mogući sadržaj:

```text
Export AppConfig konfiguracije pre produkcionog deploy-a.
```

Preporuka:

```text
Za sistemsku obradu bolje je dodati posebna metadata polja umesto oslanjanja na Description.
```

---

## 7.4 ContentType

SharePoint metadata:

```text
InternalName: ContentType
DisplayName: Content Type
TypeAsString: Computed
FieldTypeKind: 12
Group: _Hidden
```

Zaključak:

```text
Standardno SharePoint computed polje.
```

---

## 8. Očekivani export proces

Na osnovu potvrde korisnika, proces može biti opisan ovako:

```text
Canvas App / Power Automate trigger
   |
   v
Read AppConfig
   |
   v
Read šifarnici / konfiguracije
   |
   v
Generate export file
   |
   v
Save file to Exports
```

Očekivane varijante exporta:

1. Export cele `AppConfig` liste.
2. Export pojedinačne konfiguracije iz `AppConfig.Config`.
3. Export šifarnika.
4. Export prevoda.
5. Export podešavanja kolona / header-a.
6. Export konfiguracije za migraciju na drugo okruženje.
7. Backup konfiguracije pre izmena.

---

## 9. Veza sa AppConfig listom

Korisnik je potvrdio:

```text
Svi šifarnici i config JSON nalaze se u AppConfig listi.
```

Zbog toga je `Exports` logički zavisna biblioteka od `AppConfig`.

Relacija:

```text
AppConfig
   |
   | export konfiguracije / šifarnika
   v
Exports
```

Preporuka:

```text
Svaki export fajl treba da ima trag iz koje AppConfig verzije, ID-ja ili zapisa je nastao.
```

---

## 10. Trenutni nedostatak metadata modela

U dostavljenom XML-u nisu potvrđena polja kao što su:

```text
ExportType
ExportStatus
SourceList
SourceItemId
GeneratedBy
GeneratedAt
FlowRunId
Environment
Version
Checksum
```

Bez ovih polja, kasnija analiza export fajlova zavisi od:

- naziva fajla
- vremena kreiranja
- autora
- sadržaja fajla
- eventualne folder strukture

To nije dovoljno za enterprise audit.

---

## 11. Preporučeni metadata model za novu verziju

Za enterprise verziju preporučuje se proširenje biblioteke `Exports`.

| Internal name | Tip | Namena |
|---|---|---|
| `ExportType` | Choice | Tip exporta |
| `ExportStatus` | Choice | Status generisanja |
| `SourceType` | Choice | Izvor exporta |
| `SourceList` | Text | Izvorna SharePoint lista |
| `SourceItemId` | Number/Text | Izvorni item ID |
| `GeneratedByEmail` | Text | Korisnik koji je pokrenuo export |
| `GeneratedAt` | DateTime | Vreme generisanja |
| `EnvironmentName` | Text | Dev/Test/Prod |
| `AppVersion` | Text | Verzija aplikacije |
| `ConfigVersion` | Text | Verzija konfiguracije |
| `FileFormat` | Choice | JSON / CSV / XLSX / ZIP |
| `RecordCount` | Number | Broj zapisa u exportu |
| `Checksum` | Text | Hash fajla |
| `FlowRunId` | Text | Power Automate run ID |
| `ErrorMessage` | Multiple lines | Greška ako export nije uspeo |
| `RetentionCategory` | Choice | Retention/audit kategorija |

---

## 12. Predlog ExportType vrednosti

| ExportType | Opis |
|---|---|
| `AppConfigFull` | Kompletan export AppConfig liste |
| `AppConfigSingle` | Export jednog AppConfig zapisa |
| `Dictionaries` | Export šifarnika |
| `Translations` | Export prevoda |
| `ColumnHeaders` | Export konfiguracije kolona |
| `SecurityConfig` | Export konfiguracije bezbednosti |
| `MigrationPackage` | Paket za migraciju |
| `Backup` | Backup pre izmene |
| `Diagnostic` | Dijagnostički export |

---

## 13. Predlog ExportStatus vrednosti

| ExportStatus | Opis |
|---|---|
| `Requested` | Export je zatražen |
| `Processing` | Export je u toku |
| `Completed` | Export je uspešno generisan |
| `Failed` | Export nije uspeo |
| `Cancelled` | Export je otkazan |
| `Expired` | Export je zastareo |
| `Archived` | Export je arhiviran |

---

## 14. Preporučena struktura fajlova

Ako se koristi samo biblioteka bez dodatnih metadata polja, preporučuje se standardizovan naming.

Primeri:

```text
appconfig-full-prod-20260430-185913.json
appconfig-single-prod-navigation-20260430-185913.json
dictionaries-prod-20260430-185913.xlsx
translations-sr-en-prod-20260430-185913.json
migration-package-test-to-prod-20260430-185913.zip
```

Preporučeni pattern:

```text
{exportType}-{environment}-{optionalScope}-{yyyyMMdd-HHmmss}.{extension}
```

---

## 15. Preporučena folder struktura

Ako broj export fajlova raste, preporučuje se folder struktura:

```text
Exports/
├── appconfig/
│   ├── full/
│   └── single/
├── dictionaries/
├── translations/
├── migration/
├── diagnostics/
└── archive/
```

Alternativno, ako se izbegavaju folderi zbog performansi i jednostavnosti, koristiti metadata polja.

Preporuka za enterprise rešenje:

```text
Koristiti metadata polja umesto oslanjanja na folder strukturu.
```

---

## 16. Preporučeni Power Automate proces

Predloženi flow za export:

```text
Trigger
   |
   v
Validate Request
   |
   v
Read Source Data
   |
   v
Transform Data
   |
   v
Generate File
   |
   v
Create File in Exports
   |
   v
Update Metadata
   |
   v
Return File Link
```

Scope struktura:

```text
Scope - Initialize
Scope - Validate Export Request
Scope - Read AppConfig
Scope - Read Dictionaries
Scope - Generate Export Payload
Scope - Create Export File
Scope - Update Export Metadata
Scope - Respond to Power Apps
Scope - Error Handler
```

---

## 17. Power Apps uloga

Canvas aplikacija ne bi trebalo direktno da generiše kompleksne export fajlove.

Canvas aplikacija treba da:

- prikaže korisniku opcije exporta
- validira minimalni izbor
- pozove Power Automate flow
- prikaže status
- prikaže link ka generisanom fajlu

Canvas aplikacija ne treba da:

- direktno piše u `Exports` ako je proces kompleksan
- generiše finalni fajl bez backend kontrole
- čuva poslovnu logiku exporta u formuli
- tretira naziv fajla kao jedini audit podatak

---

## 18. Preporučeni request payload iz Power Apps

Primer:

```json
{
  "exportType": "AppConfigFull",
  "environmentName": "Prod",
  "requestedBy": "user@example.com",
  "requestedAt": "2026-04-30T18:59:13Z",
  "includeDictionaries": true,
  "includeTranslations": true,
  "format": "json"
}
```

---

## 19. Preporučeni response payload iz Power Automate flow-a

Primer:

```json
{
  "success": true,
  "exportType": "AppConfigFull",
  "fileName": "appconfig-full-prod-20260430-185913.json",
  "fileUrl": "https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/Exports/appconfig-full-prod-20260430-185913.json",
  "recordCount": 25,
  "flowRunId": "085848...",
  "error": null
}
```

U slučaju greške:

```json
{
  "success": false,
  "exportType": "AppConfigFull",
  "fileName": null,
  "fileUrl": null,
  "recordCount": 0,
  "flowRunId": "085848...",
  "error": "AppConfig read failed."
}
```

---

## 20. Preporučeni sadržaj AppConfig export fajla

Za JSON export preporučuje se sledeća struktura:

```json
{
  "exportMetadata": {
    "solution": "DocCentral v6.0",
    "environment": "Prod",
    "exportType": "AppConfigFull",
    "generatedAt": "2026-04-30T18:59:13Z",
    "generatedBy": "user@example.com",
    "version": "1.0"
  },
  "appConfig": [
    {
      "id": 1,
      "title": "Navigation",
      "config": {},
      "columnHeader": {}
    }
  ]
}
```

Prednost:

```text
Export je samodokumentovan i može se koristiti za restore/migration.
```

---

## 21. Security napomene

Export konfiguracije može sadržati osetljive informacije.

Moguće osetljive informacije:

- nazivi internih lista
- ID-jevi grupa
- email adrese
- konfiguracija prava
- JSON pravila
- interni endpoint-i
- nazivi flow-ova
- poslovna pravila

Zbog toga:

```text
Exports biblioteka ne treba da bude otvorena svim korisnicima.
```

Preporučeni pristup:

| Akter | Prava |
|---|---|
| Obični korisnik | Nema direktan pristup ili Read samo za svoje exporte |
| Admin | Full Control |
| Power Automate service account | Contribute/Edit |
| Audit/Compliance | Read |
| Developer/Consultant | Read/Contribute po potrebi |

---

## 22. Retention i cleanup

Export fajlovi mogu brzo zastareti.

Preporuka:

```text
Uvesti retention pravila.
```

Primer:

| Tip exporta | Retention |
|---|---|
| Diagnostic | 30 dana |
| AppConfig backup | 180 dana |
| Migration package | 365 dana |
| Audit export | prema compliance pravilima |
| Privremeni export | 7 dana |

---

## 23. Audit zahtevi

Svaki export treba da ima audit trag:

- ko je pokrenuo export
- kada je export pokrenut
- šta je eksportovano
- koliko zapisa je eksportovano
- iz kog okruženja
- koji flow run je generisao fajl
- da li je export uspeo
- gde je fajl sačuvan
- da li je fajl kasnije obrisan ili arhiviran

---

## 24. Rizici

## 24.1 Rizik nejasnog porekla fajla

Ako fajl ima samo naziv i Created date, teško je znati šta tačno sadrži.

Preporuka:

```text
Dodati metadata `ExportType`, `SourceList`, `GeneratedByEmail`, `FlowRunId`.
```

---

## 24.2 Rizik izlaganja konfiguracije

Ako export sadrži konfiguracione JSON vrednosti, može otkriti internu strukturu aplikacije.

Preporuka:

```text
Ograničiti pristup Exports biblioteci.
```

---

## 24.3 Rizik zastarelih fajlova

Korisnik može koristiti stari export kao osnovu za restore ili migraciju.

Preporuka:

```text
Dodati `ExportStatus`, `GeneratedAt`, `AppVersion`, `ConfigVersion`.
```

---

## 24.4 Rizik ručnog menjanja export fajlova

Ako korisnici imaju edit prava, mogu izmeniti export fajlove i narušiti integritet.

Preporuka:

```text
Dodati checksum i ograničiti edit prava.
```

---

## 25. Test scenariji

### Test 1 — export AppConfig konfiguracije

```text
1. Korisnik pokrene export AppConfig-a.
2. Flow čita AppConfig listu.
3. Flow generiše JSON fajl.
4. Fajl se snima u Exports.
```

Očekivanje:

```text
Fajl postoji u Exports biblioteci i ima ispravan sadržaj.
```

---

### Test 2 — export šifarnika

```text
1. Korisnik pokrene export šifarnika.
2. Flow čita relevantne konfiguracije iz AppConfig-a.
3. Flow generiše fajl.
```

Očekivanje:

```text
Export sadrži sve očekivane šifarnike.
```

---

### Test 3 — greška pri čitanju AppConfig-a

```text
1. Flow ne može da pročita AppConfig.
2. Export ne uspeva.
```

Očekivanje:

```text
Korisnik dobija kontrolisanu grešku.
Greška se loguje.
Ne kreira se nevalidan export fajl.
```

---

### Test 4 — više korisnika pokreće export istovremeno

```text
1. Dva korisnika pokreću export u isto vreme.
2. Flow generiše dva fajla.
```

Očekivanje:

```text
Nazivi fajlova su jedinstveni.
Nema prepisivanja fajlova.
```

---

### Test 5 — restore iz export fajla

```text
1. Admin preuzima export konfiguracije.
2. Koristi fajl za restore ili migraciju.
```

Očekivanje:

```text
Export sadrži dovoljno metadata da se zna poreklo i verzija fajla.
```

---

## 26. Minimalni acceptance kriterijumi za novu verziju

```text
AC-EXP-001:
Sistem mora omogućiti export AppConfig konfiguracije.

AC-EXP-002:
Sistem mora omogućiti export šifarnika.

AC-EXP-003:
Svaki export fajl mora imati jedinstven naziv.

AC-EXP-004:
Svaki export mora imati audit podatke.

AC-EXP-005:
Power Automate mora vratiti link ka fajlu nakon uspešnog exporta.

AC-EXP-006:
U slučaju greške korisnik mora dobiti kontrolisanu poruku.

AC-EXP-007:
Exports biblioteka ne sme biti otvorena za neautorizovan edit.

AC-EXP-008:
Export konfiguracije mora sadržati metadata o verziji, okruženju i vremenu generisanja.
```

---

## 27. Otvorena pitanja

| Pitanje | Status |
|---|---|
| Koji je tačan naziv flow-a za export? | NEPOZNATO |
| Da li se export pokreće iz Canvas aplikacije? | NEPOZNATO |
| Da li se export može pokrenuti ručno iz SharePoint-a? | NEPOZNATO |
| Da li postoji scheduled export? | NEPOZNATO |
| Koji format se koristi za export: JSON, CSV, XLSX ili ZIP? | NEPOZNATO |
| Da li jedan export fajl sadrži sve konfiguracije ili postoje posebni fajlovi? | NEPOZNATO |
| Da li se export koristi za migraciju između okruženja? | NEPOZNATO |
| Da li postoji import/restore funkcija? | NEPOZNATO |
| Da li su export fajlovi dostupni krajnjim korisnicima? | NEPOZNATO |
| Da li postoji retention policy? | NEPOZNATO |
| Da li postoji audit/log za export procese? | NEPOZNATO |

---

## 28. Zaključak

`Exports` je SharePoint document library koja u DocCentral v6.0 služi za export šifarnika i konfiguracije iz `AppConfig` liste.

Trenutno su potvrđena samo osnovna SharePoint document library polja:

```text
FileLeafRef
Title
_ExtendedDescription
ContentType
```

Za enterprise verziju treba dodati metadata koja omogućava:

- jasnu klasifikaciju exporta
- audit
- troubleshooting
- kontrolu pristupa
- verzionisanje
- restore/migration scenario
- lifecycle management

Najvažnija preporuka:

```text
Export fajl ne sme biti samo fajl sa imenom; mora imati metadata i audit trag.
```

---

## 29. Povezani dokumenti

```text
README.md
docs/03-sharepoint-data-model.md
docs/04-appconfig-analysis.md
docs/05-document-libraries.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
docs/10-known-issues-technical-debt.md
docs/11-enterprise-improvements.md
data-model/appconfig.md
data-model/shared-documents.md
data-model/email-documents.md
architecture/current-architecture.md
backlog/known-issues.md
backlog/open-questions.md
```
