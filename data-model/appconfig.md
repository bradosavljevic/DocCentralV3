# Data Model — AppConfig

## 1. Svrha dokumenta

Ovaj dokument opisuje SharePoint listu:

```text
AppConfig
```

Lista `AppConfig` predstavlja centralnu konfiguracionu listu aplikacije **DocCentral v6.0**.

Na osnovu dostavljenih SharePoint REST XML metadata odgovora, ova lista sadrži tekstualna i višelinijska polja namenjena čuvanju konfiguracionih vrednosti, kolonskih podešavanja i potencijalno JSON konfiguracija koje aplikacija koristi.

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Postojanje liste `AppConfig` | POTVRĐENO |
| SharePoint scope liste | POTVRĐENO |
| Ključna polja liste | POTVRĐENO |
| Konkretan sadržaj `Config` polja | NEPOZNATO |
| Konkretan sadržaj `ColumnHeader` polja | NEPOZNATO |
| Način učitavanja u Canvas aplikaciju | PRETPOSTAVKA |
| Flow za export konfiguracije | POTVRĐENO funkcionalno, naziv flow-a nije dostavljen |

---

## 3. SharePoint lokacija

Site:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

Lista:

```text
/sites/DocumentCentralv6.0/Lists/AppConfig
```

List GUID iz dostavljenog XML-a:

```text
5b76665b-28c3-4bdb-9426-345045620a26
```

REST endpoint obrazac:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/Web/Lists(guid'5b76665b-28c3-4bdb-9426-345045620a26')
```

---

## 4. Identifikovana polja

### 4.1 Poslovna / konfiguraciona polja

| Internal name | Display name | Tip | Required | Read only | Filterable | Napomena |
|---|---|---|---|---|---|---|
| `Title` | Title | Single line of text | Ne | Ne | Da | Naziv konfiguracione stavke |
| `Config` | Config | Multiple lines of text | Ne | Ne | Ne | Verovatno JSON konfiguracija |
| `ColumnHeader` | ColumnHeader | Multiple lines of text | Ne | Ne | Ne | Verovatno konfiguracija kolona / headera |

---

### 4.2 Sistemska SharePoint polja

| Internal name | Display name | Tip | Required | Read only |
|---|---|---|---|---|
| `ID` | ID | Counter | Ne | Da |
| `Created` | Created | Date and Time | Ne | Da |
| `Modified` | Modified | Date and Time | Ne | Da |
| `Author` | Created By | Person or Group | Ne | Da |
| `Editor` | Modified By | Person or Group | Ne | Da |
| `Attachments` | Attachments | Attachments | Ne | Ne |
| `_UIVersionString` | Version | Single line of text | Ne | Da |
| `ComplianceAssetId` | Compliance Asset Id | Single line of text | Ne | Da |
| `_ColorTag` | Color Tag | Single line of text | Ne | Da |
| `ContentType` | Content Type | Computed | Ne | Ne |

---

## 5. Detalji ključnih polja

## 5.1 Title

SharePoint metadata:

```text
InternalName: Title
DisplayName: Title
TypeAsString: Text
FieldTypeKind: 2
MaxLength: 255
Required: false
ReadOnlyField: false
EnforceUniqueValues: false
Indexed: false
```

Namena:

```text
Identifikator konfiguracione stavke.
```

Primeri mogućih vrednosti:

```text
GlobalConfig
Translations
ColumnHeaders
DocumentTypes
SecurityRoles
FeatureFlags
```

Status:

```text
POTVRĐENO: polje postoji.
NEPOZNATO: stvarne vrednosti iz liste nisu dostavljene.
```

---

## 5.2 Config

SharePoint metadata:

```text
InternalName: Config
DisplayName: Config
TypeAsString: Note
FieldTypeKind: 3
RichText: false
AppendOnly: false
Filterable: false
NumberOfLines: 6
Required: false
ReadOnlyField: false
```

Namena:

```text
Centralno polje za tekstualnu ili JSON konfiguraciju aplikacije.
```

Pretpostavljena upotreba:

- globalna podešavanja aplikacije
- JSON objekti za inicijalizaciju kolekcija
- podešavanja šifarnika
- feature flags
- prevodi
- pravila prikaza
- pravila validacije
- podešavanja za export

Status:

```text
PRETPOSTAVKA
```

Razlog:

```text
XML potvrđuje da polje postoji i da je Multiple lines of text, ali konkretan sadržaj nije dostavljen.
```

---

## 5.3 ColumnHeader

SharePoint metadata:

```text
InternalName: ColumnHeader
DisplayName: ColumnHeader
TypeAsString: Note
FieldTypeKind: 3
RichText: false
AppendOnly: false
Filterable: false
NumberOfLines: 6
Required: false
ReadOnlyField: false
```

Namena:

```text
Verovatno čuva konfiguraciju prikaza kolona, naziva kolona ili headera za export / UI.
```

Pretpostavljena upotreba:

- nazivi kolona za prikaz u aplikaciji
- nazivi kolona za export
- mapiranje internal name -> display name
- višejezični headeri
- redosled kolona
- vidljivost kolona

Status:

```text
PRETPOSTAVKA
```

Razlog:

```text
Naziv polja ukazuje na konfiguraciju kolonskih headera, ali konkretan sadržaj nije dostavljen.
```

---

## 6. Poslovna uloga liste AppConfig

Lista `AppConfig` najverovatnije služi kao centralni configuration store za DocCentral aplikaciju.

Korisnik je potvrdio:

```text
Svi šifarnici i config JSON nalaze se u AppConfig listi.
```

To znači da `AppConfig` nije obična pomoćna lista, već važan deo aplikacione logike.

Ova lista verovatno zamenjuje više posebnih SharePoint šifarnika i omogućava da se konfiguracija menja bez izmene Canvas aplikacije.

---

## 7. Predloženi logički model

Iako SharePoint metadata potvrđuje samo nekoliko kolona, preporučeni logički model korišćenja je:

| Kolona | Namena |
|---|---|
| `Title` | Naziv konfiguracione grupe |
| `Config` | JSON konfiguracija |
| `ColumnHeader` | JSON konfiguracija kolona / prikaza / exporta |

Primer:

```json
{
  "key": "DocumentTypes",
  "version": "1.0",
  "enabled": true,
  "items": [
    {
      "code": "INVOICE",
      "labelSr": "Faktura",
      "labelEn": "Invoice",
      "active": true
    }
  ]
}
```

---

## 8. Preporučena struktura Config JSON-a

Za novu enterprise verziju preporučuje se standardizovan JSON format.

### 8.1 Osnovna struktura

```json
{
  "schemaVersion": "1.0",
  "configKey": "DocumentTypes",
  "description": "Document type configuration",
  "isActive": true,
  "lastUpdatedBy": "admin@domain.com",
  "lastUpdatedAt": "2026-04-30T00:00:00Z",
  "items": []
}
```

---

### 8.2 Primer za šifarnik tipova dokumenata

```json
{
  "schemaVersion": "1.0",
  "configKey": "DocumentTypes",
  "isActive": true,
  "items": [
    {
      "code": "CONTRACT",
      "labelSr": "Ugovor",
      "labelEn": "Contract",
      "requiresAttachment": true,
      "requiresApproval": true,
      "active": true
    },
    {
      "code": "INVOICE",
      "labelSr": "Faktura",
      "labelEn": "Invoice",
      "requiresAttachment": true,
      "requiresApproval": false,
      "active": true
    }
  ]
}
```

---

### 8.3 Primer za prevode

```json
{
  "schemaVersion": "1.0",
  "configKey": "Translations",
  "isActive": true,
  "items": [
    {
      "key": "Home.Title",
      "sr": "Početna",
      "en": "Home"
    },
    {
      "key": "Document.Register",
      "sr": "Zavedite dokument",
      "en": "Register document"
    }
  ]
}
```

---

### 8.4 Primer za feature flags

```json
{
  "schemaVersion": "1.0",
  "configKey": "FeatureFlags",
  "isActive": true,
  "items": [
    {
      "key": "EnableEmailIntake",
      "enabled": true
    },
    {
      "key": "EnableExports",
      "enabled": true
    },
    {
      "key": "EnableAdvancedAudit",
      "enabled": false
    }
  ]
}
```

---

## 9. Preporučena struktura ColumnHeader JSON-a

Ako se `ColumnHeader` koristi za export i UI prikaz, preporučeni format je:

```json
{
  "schemaVersion": "1.0",
  "configKey": "DocumentExportColumns",
  "columns": [
    {
      "internalName": "DelovodniBroj",
      "labelSr": "Delovodni broj",
      "labelEn": "Registry number",
      "order": 1,
      "visible": true,
      "export": true
    },
    {
      "internalName": "FileLeafRef",
      "labelSr": "Naziv dokumenta",
      "labelEn": "Document name",
      "order": 2,
      "visible": true,
      "export": true
    }
  ]
}
```

---

## 10. Učitavanje AppConfig u Canvas aplikaciju

Potvrđeno:

```text
Aplikacija je Canvas App.
Svi šifarnici i config JSON nalaze se u AppConfig listi.
```

Preporučeni obrazac:

```powerfx
ClearCollect(
    colAppConfig,
    AppConfig
);
```

Ako se konfiguracija vraća preko Power Automate flow-a, preporučeni obrazac je:

```powerfx
ClearCollect(
    colAppConfig,
    Table(ParseJSON(Flow_GetAppConfig.Run().result))
);
```

Napomena:

```text
Konkretan postojeći Power Fx kod nije dostavljen, pa je ovo preporučeni obrazac, ne potvrđena implementacija.
```

---

## 11. Parsiranje JSON konfiguracije u Power Apps

Power Apps može koristiti `ParseJSON`, ali je potrebno standardizovati strukturu.

Primer:

```powerfx
Set(
    gblAppConfigRaw,
    LookUp(colAppConfig, Title = "GlobalConfig").Config
);

Set(
    gblAppConfigJson,
    ParseJSON(gblAppConfigRaw)
);
```

Za čitanje pojedinačne vrednosti:

```powerfx
Set(
    gblDefaultLanguage,
    Text(gblAppConfigJson.defaultLanguage)
);
```

Za niz:

```powerfx
ClearCollect(
    colDocumentTypes,
    ForAll(
        Table(ParseJSON(LookUp(colAppConfig, Title = "DocumentTypes").Config).items),
        {
            Code: Text(Value.code),
            LabelSr: Text(Value.labelSr),
            LabelEn: Text(Value.labelEn),
            Active: Boolean(Value.active)
        }
    )
);
```

---

## 12. Validacija JSON konfiguracije

Trenutni rizik:

```text
Ako korisnik ručno izmeni JSON u AppConfig listi i napravi sintaksnu grešku, Canvas aplikacija ili flow mogu prestati da rade.
```

Preporuka:

- svaka JSON konfiguracija mora imati `schemaVersion`
- svaka konfiguracija mora imati `configKey`
- flow za export / import mora validirati JSON
- treba imati backup prethodne verzije
- treba imati status `Active`
- treba imati kontrolu ko sme da menja konfiguraciju

---

## 13. Predložena dodatna polja za enterprise verziju

Postojeća polja su minimalna.

Za enterprise verziju preporučuje se proširenje liste ili nova verzija liste `AppConfigV2`.

Predložena polja:

| Internal name | Tip | Namena |
|---|---|---|
| `ConfigKey` | Text | Stabilni ključ konfiguracije |
| `ConfigType` | Choice | Tip konfiguracije |
| `ConfigJson` | Multiple lines | JSON sadržaj |
| `SchemaVersion` | Text | Verzija JSON šeme |
| `IsActive` | Yes/No | Da li je aktivno |
| `Environment` | Choice | Dev / Test / Prod |
| `Language` | Choice | sr / en / all |
| `SortOrder` | Number | Redosled |
| `Description` | Multiple lines | Opis konfiguracije |
| `LastValidatedAt` | DateTime | Kada je JSON validiran |
| `ValidationStatus` | Choice | Valid / Invalid / Warning |
| `ValidationMessage` | Multiple lines | Poruka validacije |

---

## 14. Preporučeni tipovi konfiguracije

| ConfigType | Namena |
|---|---|
| `GlobalSettings` | Globalna podešavanja aplikacije |
| `Translations` | Prevodi |
| `DocumentTypes` | Tipovi dokumenata |
| `ColumnHeaders` | Headeri za UI i export |
| `ExportConfig` | Export pravila |
| `EmailIntakeConfig` | Pravila za email dokumente |
| `NumberingConfig` | Pravila delovodnog broja |
| `SecurityRoles` | Role i mapiranja |
| `FeatureFlags` | Uključivanje / isključivanje funkcionalnosti |
| `ValidationRules` | Poslovne validacije |

---

## 15. Export konfiguracije

Korisnik je potvrdio:

```text
Exporti se koriste za export šifarnika i konfiguracije iz AppConfig-a.
```

Zaključak:

```text
AppConfig je izvor za export konfiguracionih podataka.
```

Povezana biblioteka:

```text
Exports
```

Preporučeni flow obrazac:

```text
Power Apps / Manual trigger
   |
   v
Read AppConfig items
   |
   v
Build JSON / Excel / CSV
   |
   v
Create file in Exports library
   |
   v
Return export file link
```

Status:

```text
POTVRĐENO: funkcionalna namena exporta.
NEPOZNATO: naziv postojećeg export flow-a.
```

---

## 16. Bezbednosni model za AppConfig

Zbog značaja liste `AppConfig`, pristup ne sme biti široko otvoren.

Preporuka:

| Grupa korisnika | Pristup |
|---|---|
| Obični korisnici | Read |
| Referenti / korisnici aplikacije | Read |
| Administratori aplikacije | Edit |
| Service account za flow-ove | Edit / Full Control |
| Developeri | Po potrebi, kontrolisano |
| External users | Bez pristupa, osim ako postoji poseban razlog |

Rizik:

```text
Ako običan korisnik može da menja AppConfig, može nenamerno ili namerno promeniti ponašanje aplikacije.
```

---

## 17. Audit zahtevi

Za svaku promenu konfiguracije treba znati:

- ko je promenio
- kada je promenio
- šta je promenio
- prethodna vrednost
- nova vrednost
- da li je JSON validan
- da li je promena objavljena u produkciji

Minimalni audit može koristiti SharePoint version history.

Enterprise preporuka:

```text
Uvesti posebnu AppConfigAudit listu.
```

Predložena audit polja:

| Polje | Tip |
|---|---|
| `ConfigKey` | Text |
| `ChangedByEmail` | Text |
| `ChangedAt` | DateTime |
| `OldValue` | Multiple lines |
| `NewValue` | Multiple lines |
| `ValidationStatus` | Choice |
| `CorrelationId` | Text |
| `ChangeReason` | Multiple lines |

---

## 18. Rizici

| Rizik | Opis | Uticaj |
|---|---|---|
| Nevalidan JSON | Ručna izmena može pokvariti aplikaciju | Visok |
| Nema schema version | Teško se upravlja promenama | Srednji / visok |
| Previše logike u jednoj koloni | Teško održavanje | Srednji |
| Nema role-based edit prava | Korisnici mogu promeniti konfiguraciju | Visok |
| Nema audit log | Teško otkriti uzrok greške | Visok |
| Nema environment razdvajanja | Dev/Test/Prod konfiguracije se mogu pomešati | Visok |
| Nema backup konfiguracije | Teško vraćanje prethodnog stanja | Srednji / visok |

---

## 19. Preporuke za novu verziju

Za novu enterprise verziju DocCentral-a preporučuje se:

1. Standardizovati `AppConfig` JSON strukturu.
2. Uvesti `schemaVersion`.
3. Uvesti `configKey`.
4. Uvesti validaciju JSON-a pre čuvanja.
5. Ograničiti edit pristup samo administratorima i service account-u.
6. Uvesti audit promene konfiguracije.
7. Razdvojiti konfiguraciju po okruženjima.
8. Napraviti export i import mehanizam.
9. Napraviti backup pre svake izmene.
10. U Power Apps učitavati konfiguraciju kontrolisano i sa fallback vrednostima.

---

## 20. Otvorena pitanja

| Pitanje | Status |
|---|---|
| Koji sve `Title` zapisi postoje u AppConfig listi? | NEPOZNATO |
| Šta je tačan sadržaj `Config` polja? | NEPOZNATO |
| Šta je tačan sadržaj `ColumnHeader` polja? | NEPOZNATO |
| Da li Canvas aplikacija direktno čita AppConfig ili preko flow-a? | NEPOZNATO |
| Koji flow radi export AppConfig konfiguracije? | NEPOZNATO |
| Da li postoji import AppConfig konfiguracije? | NEPOZNATO |
| Da li AppConfig ima version history uključen? | NEPOZNATO |
| Ko ima edit prava nad AppConfig listom? | NEPOZNATO |
| Da li postoji validacija JSON-a? | NEPOZNATO |
| Da li postoji AppConfig audit? | NEPOZNATO |

---

## 21. Zaključak

`AppConfig` je ključna lista za konfiguraciju DocCentral aplikacije.

Potvrđeno je da:

```text
Svi šifarnici i config JSON nalaze se u AppConfig listi.
```

To znači da nova verzija rešenja mora tretirati `AppConfig` kao centralni application configuration store.

Najvažnija preporuka:

```text
AppConfig treba formalizovati kao verzionisani, validirani i auditovani configuration layer.
```

Bez toga postoji rizik da ručne izmene konfiguracije izazovu greške u aplikaciji, flow-ovima, exportima ili poslovnim pravilima.

---

## 22. Povezani dokumenti

```text
README.md
docs/03-sharepoint-data-model.md
docs/04-appconfig-analysis.md
docs/11-enterprise-improvements.md
architecture/current-architecture.md
architecture/target-architecture.md
data-model/rezervisani-brojevi.md
data-model/shared-documents.md
data-model/email-documents.md
data-model/exports.md
backlog/known-issues.md
backlog/open-questions.md
```
