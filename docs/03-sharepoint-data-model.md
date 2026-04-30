# 03 — SharePoint Data Model

## 1. Svrha dokumenta

Ovaj dokument opisuje trenutno identifikovani SharePoint data model za rešenje **DocCentral v6.0**.

Cilj dokumenta je da prikaže:

- identifikovane SharePoint liste i biblioteke
- potvrđena polja i njihove tipove
- poslovnu ulogu svake liste/biblioteke
- rizike trenutnog modela
- preporuke za enterprise unapređenje
- oblasti koje su nepoznate i zahtevaju dodatnu proveru

Dokument je zasnovan na dostavljenim SharePoint REST XML metadata odgovorima.

---

## 2. Identifikovani SharePoint site

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

REST API base:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 3. Pregled identifikovanih lista i biblioteka

| Naziv | Tip | Scope | Status |
|---|---|---|---|
| `AppConfig` | SharePoint lista | `/sites/DocumentCentralv6.0/Lists/AppConfig` | Potvrđeno |
| `RezervisaniBrojevi` | SharePoint lista | `/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi` | Potvrđeno |
| `Shared Documents` | Document library | `/sites/DocumentCentralv6.0/Shared Documents` | Potvrđeno |
| `EmailDocuments` | Document library | `/sites/DocumentCentralv6.0/EmailDocuments` | Potvrđeno |
| `Exports` | Document library | `/sites/DocumentCentralv6.0/Exports` | Potvrđeno |

---

## 4. `AppConfig`

### 4.1 Opis

`AppConfig` je SharePoint lista koja predstavlja centralno mesto za konfiguraciju aplikacije.

Na osnovu naziva i identifikovanih kolona, lista se koristi ili je namenjena za čuvanje konfiguracionih vrednosti koje aplikacija i/ili flow-ovi koriste tokom rada.

### 4.2 Scope

```text
/sites/DocumentCentralv6.0/Lists/AppConfig
```

### 4.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Napomena |
|---|---|---|---|---|---|
| `Title` | Title | Single line of text | Ne | Ne | Osnovni naziv zapisa |
| `Config` | Config | Multiple lines of text | Ne | Ne | Verovatno JSON konfiguracija |
| `ColumnHeader` | ColumnHeader | Multiple lines of text | Ne | Ne | Verovatno konfiguracija kolona/prikaza |
| `ID` | ID | Counter | Ne | Da | SharePoint sistemski ID |
| `Created` | Created | Date and Time | Ne | Da | Sistemsko polje |
| `Modified` | Modified | Date and Time | Ne | Da | Sistemsko polje |
| `Author` | Created By | Person or Group | Ne | Da | Sistemsko polje |
| `Editor` | Modified By | Person or Group | Ne | Da | Sistemsko polje |
| `Attachments` | Attachments | Attachments | Ne | Ne | Sistemsko attachment polje |
| `ContentType` | Content Type | Computed | Ne | Ne | Sistemsko polje |

### 4.4 Potvrđeni tehnički detalji

Polje `Config`:

```text
InternalName: Config
TypeAsString: Note
TypeDisplayName: Multiple lines of text
RichText: false
AppendOnly: false
```

Polje `ColumnHeader`:

```text
InternalName: ColumnHeader
TypeAsString: Note
TypeDisplayName: Multiple lines of text
RichText: false
AppendOnly: false
```

### 4.5 Pretpostavljena poslovna uloga

`AppConfig` može sadržati:

- globalna podešavanja aplikacije
- konfiguraciju delovodnih knjiga
- konfiguraciju tipova dokumenata
- prevode
- UI podešavanja
- konfiguraciju kolona
- korisnička ili grupna pravila
- parametre za flow-ove
- poslovna pravila za zavođenje dokumenata

Status: **pretpostavka**

Razlog: XML metadata potvrđuje strukturu liste, ali ne potvrđuje sadržaj zapisa.

### 4.6 Rizici

| Rizik | Opis | Prioritet |
|---|---|---|
| Nedokumentovan JSON | Ako `Config` sadrži JSON bez šeme, održavanje je teško | Visok |
| Nedostatak validacije | SharePoint Note polje ne validira JSON strukturu | Visok |
| Ručna izmena konfiguracije | Ručno menjanje JSON-a može pokvariti aplikaciju | Visok |
| Nema verzionisane šeme | Promena formata može polomiti aplikaciju ili flow | Visok |
| Nema tipizacije konfiguracije | Nije jasno šta svaki zapis predstavlja | Srednji |

### 4.7 Preporuke

Za enterprise verziju preporučuje se:

- definisati JSON šemu za svaki tip konfiguracije
- dodati kolonu `ConfigType`
- dodati kolonu `IsActive`
- dodati kolonu `Version`
- dodati kolonu `Environment`
- dodati kolonu `LastValidatedOn`
- dodati kolonu `ValidationStatus`
- uvesti flow za validaciju konfiguracije pre izmene
- čuvati primer JSON konfiguracije u GitHub dokumentaciji

Predložena dodatna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `ConfigType` | Choice/Text | Tip konfiguracije |
| `IsActive` | Yes/No | Aktivna konfiguracija |
| `Version` | Single line of text | Verzija konfiguracije |
| `Environment` | Choice/Text | Dev/Test/Prod |
| `Description` | Multiple lines of text | Opis konfiguracije |
| `SchemaVersion` | Single line of text | Verzija JSON šeme |
| `ValidationStatus` | Choice | Valid/Invalid/Not checked |
| `LastValidatedOn` | Date and Time | Datum poslednje validacije |

---

## 5. `RezervisaniBrojevi`

### 5.1 Opis

`RezervisaniBrojevi` je SharePoint lista koja je najverovatnije namenjena za rezervaciju delovodnih brojeva.

Ova lista je kritična zbog zahteva da više korisnika istovremeno može da zavodi dokumente, ali da sistem nikada ne sme dodeliti isti delovodni broj za dva dokumenta.

### 5.2 Scope

```text
/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi
```

### 5.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Napomena |
|---|---|---|---|---|---|
| `Title` | Title | Single line of text | Ne | Ne | Osnovni naziv zapisa |
| `RezervisaniBroj` | Rezervisani broj | Number | Ne | Ne | Rezervisani broj |
| `DatumRezervacije` | DatumRezervacije | Date and Time | Ne | Ne | Datum rezervacije |
| `ID` | ID | Counter | Ne | Da | SharePoint sistemski ID |
| `Created` | Created | Date and Time | Ne | Da | Sistemsko polje |
| `Modified` | Modified | Date and Time | Ne | Da | Sistemsko polje |
| `Author` | Created By | Person or Group | Ne | Da | Sistemsko polje |
| `Editor` | Modified By | Person or Group | Ne | Da | Sistemsko polje |
| `Attachments` | Attachments | Attachments | Ne | Ne | Sistemsko attachment polje |

### 5.4 Potvrđeni tehnički detalji

Polje `RezervisaniBroj`:

```text
InternalName: RezervisaniBroj
TypeAsString: Number
Decimals: 0
CommaSeparator: true
```

Polje `DatumRezervacije`:

```text
InternalName: DatumRezervacije
TypeAsString: DateTime
Format: DateOnly
```

### 5.5 Pretpostavljena poslovna uloga

Lista može služiti za:

- privremenu rezervaciju sledećeg broja
- sprečavanje duplikata
- evidenciju pokušaja rezervacije
- istoriju rezervisanih brojeva

Status: **pretpostavka**

Razlog: naziv liste i polja ukazuju na ovu namenu, ali nije dostavljen flow koji koristi listu.

### 5.6 Kritični nedostaci trenutnog modela

Na osnovu metadata odgovora nisu potvrđena sledeća polja koja su bitna za enterprise numbering servis:

| Potrebno polje | Zašto je potrebno |
|---|---|
| `DelovodnaKnjiga` | Brojevi se često vode po knjizi/registru |
| `Godina` | Brojevi se najčešće resetuju ili grupišu po godini |
| `Status` | Rezervisan, iskorišćen, poništen, istekao |
| `DocumentId` | Veza ka finalnom dokumentu |
| `RequestId` | Idempotency i ponovljeni pokušaji |
| `ReservedBy` | Korisnik koji je pokrenuo rezervaciju |
| `ReservedByEmail` | Email korisnika |
| `ReservedAt` | Tačan timestamp rezervacije |
| `ExpiresAt` | Istek privremene rezervacije |
| `FinalDelovodniBroj` | Formatirani poslovni broj |
| `ErrorMessage` | Greška ako rezervacija ne uspe |
| `CorrelationId` | Praćenje kroz flow run-ove |

### 5.7 Rizici

| Rizik | Opis | Prioritet |
|---|---|---|
| Dupli broj | Ako nema atomic kontrole, dva korisnika mogu dobiti isti broj | Kritično |
| Nema statusa rezervacije | Ne zna se da li je broj iskorišćen ili napušten | Visok |
| Nema veze sa dokumentom | Teško dokazati koji broj pripada kom dokumentu | Visok |
| Nema idempotency ključa | Ponovni poziv flow-a može kreirati novi broj | Visok |
| Nema audit kolona | Teško istražiti greške | Visok |
| Nema potvrđen unique constraint | SharePoint možda neće sprečiti duplikate | Kritično |

### 5.8 Obavezna enterprise preporuka

Za novu verziju, `RezervisaniBrojevi` ne treba tretirati samo kao običnu pomoćnu listu, već kao deo **numbering servisa**.

Minimalni enterprise model:

| Internal name | Tip | Obavezno | Namena |
|---|---|---|---|
| `Title` | Single line of text | Da | Jedinstveni ključ rezervacije |
| `DelovodnaKnjiga` | Single line of text / Lookup | Da | Knjiga/registar |
| `Godina` | Number | Da | Godina |
| `RezervisaniBroj` | Number | Da | Numerički broj |
| `FinalDelovodniBroj` | Single line of text | Da | Kompletan formatirani broj |
| `Status` | Choice | Da | Reserved/Used/Cancelled/Expired/Error |
| `RequestId` | Single line of text | Da | Idempotency key |
| `DocumentItemId` | Number | Ne | ID dokumenta/predmeta |
| `DocumentLibraryItemId` | Number | Ne | ID fajla u biblioteci |
| `ReservedByEmail` | Single line of text | Da | Korisnik |
| `ReservedAt` | Date and Time | Da | Vreme rezervacije |
| `UsedAt` | Date and Time | Ne | Vreme finalnog korišćenja |
| `ExpiresAt` | Date and Time | Ne | Istek rezervacije |
| `CorrelationId` | Single line of text | Da | Trace ID |
| `ErrorMessage` | Multiple lines of text | Ne | Greška |

Unique constraints:

- `FinalDelovodniBroj` mora biti unique
- `RequestId` treba biti unique ili bar logički jedinstven
- kombinacija `DelovodnaKnjiga + Godina + RezervisaniBroj` mora biti jedinstvena

Napomena:

SharePoint ne podržava kompozitni unique constraint kao relaciona baza. Zato treba koristiti formatirani ključ u jednoj koloni, na primer:

```text
DK-2026-000001
```

i na toj koloni uključiti unique values.

---

## 6. `Shared Documents`

### 6.1 Opis

`Shared Documents` je glavna SharePoint document library za čuvanje dokumenata.

### 6.2 Scope

```text
/sites/DocumentCentralv6.0/Shared Documents
```

### 6.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Napomena |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Naziv fajla |
| `Title` | Title | Single line of text | Ne | Ne | Sealed field |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Rich text description |
| `DelovodniBroj` | DelovodniBroj | Single line of text | Ne | Ne | Poslovni delovodni broj |
| `EdokumentID` | EdokumentID | Single line of text | Ne | Ne | ID elektronskog dokumenta |
| `Edokument` | Edokument | Yes/No | Ne | Ne | Oznaka e-dokumenta |
| `Attachment` | Attachment | Yes/No | Ne | Ne | Oznaka priloga |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Sistemski/media metadata |
| `ContentType` | Content Type | Computed | Ne | Ne | Sistemsko polje |

### 6.4 Potvrđeni tehnički detalji

Polje `DelovodniBroj`:

```text
InternalName: DelovodniBroj
TypeAsString: Text
MaxLength: 255
Indexed: false
EnforceUniqueValues: false
```

Polje `Edokument`:

```text
InternalName: Edokument
TypeAsString: Boolean
DefaultValue: 0
```

Polje `Attachment`:

```text
InternalName: Attachment
TypeAsString: Boolean
DefaultValue: 0
```

### 6.5 Kritična napomena za `DelovodniBroj`

Iz dostavljenog XML-a je vidljivo da:

```text
Indexed: false
EnforceUniqueValues: false
```

To znači da na osnovu dostavljenih metadata odgovora nije potvrđena zaštita od duplog delovodnog broja u biblioteci `Shared Documents`.

Ovo je kritičan rizik.

### 6.6 Rizici

| Rizik | Opis | Prioritet |
|---|---|---|
| `DelovodniBroj` nije unique | Biblioteka može prihvatiti duplikate | Kritično |
| `DelovodniBroj` nije indexed | Pretraga i provera mogu biti spori kod velikog broja dokumenata | Visok |
| Nejasna veza sa rezervisanim brojem | Nije potvrđena veza ka `RezervisaniBrojevi` | Visok |
| Nema potvrđen status dokumenta | Nije vidljivo stanje dokumenta | Srednji |
| Nema potvrđen audit model | Teško praćenje promena | Visok |

### 6.7 Preporuke

Za enterprise verziju:

- uključiti indexing na `DelovodniBroj`
- uključiti unique values na `DelovodniBroj`, ako je to finalni jedinstveni broj
- razmotriti posebnu kolonu `DelovodniBrojKey`
- dodati vezu ka rezervaciji broja
- dodati status dokumenta
- dodati audit / correlation kolone
- jasno razdvojiti dokument od priloga
- standardizovati e-dokument metadata

Predložena dodatna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `DelovodniBrojKey` | Single line of text | Unique tehnički ključ |
| `RezervacijaBrojaId` | Number / Lookup | Veza ka rezervaciji |
| `StatusDokumenta` | Choice | Draft/Zaveden/Arhiviran/Poništen |
| `DocumentType` | Choice / Lookup | Tip dokumenta |
| `DocumentDirection` | Choice | Ulazni/Izlazni/Interni |
| `RegisteredAt` | Date and Time | Vreme zavođenja |
| `RegisteredByEmail` | Single line of text | Ko je zaveo |
| `CorrelationId` | Single line of text | Trace kroz flow |
| `SourceSystem` | Choice/Text | Manual/Email/API/Import |
| `ParentDocumentId` | Number | Veza za priloge |

---

## 7. `EmailDocuments`

### 7.1 Opis

`EmailDocuments` je document library koja je verovatno namenjena za dokumente dobijene emailom.

### 7.2 Scope

```text
/sites/DocumentCentralv6.0/EmailDocuments
```

### 7.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Napomena |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Naziv fajla |
| `Title` | Title | Single line of text | Ne | Ne | Title metadata |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Opis |
| `PosiljalacEmail` | PosiljalacEmail | Single line of text | Ne | Ne | Email pošiljaoca |
| `Posiljalac` | Posiljalac | Single line of text | Ne | Ne | Naziv pošiljaoca |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Media metadata |
| `ContentType` | Content Type | Computed | Ne | Ne | Sistemsko polje |

### 7.4 Pretpostavljena poslovna uloga

Biblioteka verovatno služi za:

- prijem email attachment-a
- čuvanje email dokumenata pre zavođenja
- čuvanje pošiljaoca i email adrese
- ulaznu obradu dokumenata

Status: **pretpostavka**

### 7.5 Rizici

| Rizik | Opis | Prioritet |
|---|---|---|
| Nema statusa obrade | Nije jasno da li je email dokument obrađen | Visok |
| Nema veze ka finalnom dokumentu | Nije jasno da li je email dokument zaveden | Visok |
| Nema email message ID | Teško sprečiti duplu obradu istog emaila | Visok |
| Nema attachment ID | Teško pratiti pojedinačne priloge | Srednji |
| Nema error log | Teško analizirati neuspešne obrade | Srednji |

### 7.6 Preporuke

Dodati polja:

| Internal name | Tip | Namena |
|---|---|---|
| `EmailMessageId` | Single line of text | Jedinstveni ID email poruke |
| `EmailConversationId` | Single line of text | Thread/conversation ID |
| `AttachmentId` | Single line of text | ID attachment-a |
| `ProcessingStatus` | Choice | New/Processed/Error/Ignored |
| `ProcessedAt` | Date and Time | Datum obrade |
| `ProcessedByFlowRunId` | Single line of text | Flow run |
| `TargetDocumentId` | Number | Veza ka zavedenom dokumentu |
| `ErrorMessage` | Multiple lines of text | Greška obrade |

---

## 8. `Exports`

### 8.1 Opis

`Exports` je document library za čuvanje exportovanih fajlova.

### 8.2 Scope

```text
/sites/DocumentCentralv6.0/Exports
```

### 8.3 Potvrđena polja

| Internal name | Display name | Tip | Required | Read-only | Napomena |
|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Naziv fajla |
| `Title` | Title | Single line of text | Ne | Ne | Title metadata |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Opis |
| `ContentType` | Content Type | Computed | Ne | Ne | Sistemsko polje |

### 8.4 Pretpostavljena poslovna uloga

Biblioteka verovatno služi za:

- export izveštaje
- generisane dokumente
- Excel/CSV/PDF izlaze
- privremene fajlove za preuzimanje

Status: **pretpostavka**

### 8.5 Rizici

| Rizik | Opis | Prioritet |
|---|---|---|
| Nema tipa exporta | Nije jasno šta fajl predstavlja | Srednji |
| Nema statusa | Nije jasno da li je export uspešan | Srednji |
| Nema retention pravila | Export fajlovi mogu rasti bez kontrole | Srednji |
| Nema korisnika | Nije jasno ko je tražio export | Srednji |

### 8.6 Preporuke

Dodati polja:

| Internal name | Tip | Namena |
|---|---|---|
| `ExportType` | Choice/Text | Tip exporta |
| `RequestedByEmail` | Single line of text | Ko je pokrenuo export |
| `RequestedAt` | Date and Time | Vreme zahteva |
| `GeneratedAt` | Date and Time | Vreme generisanja |
| `ExportStatus` | Choice | Pending/Generated/Error/Expired |
| `FlowRunId` | Single line of text | Flow run |
| `ExpiresAt` | Date and Time | Rok čuvanja fajla |
| `ErrorMessage` | Multiple lines of text | Greška |

---

## 9. Sistemska polja

U svim listama i bibliotekama postoje standardna SharePoint sistemska polja, među kojima su:

- `ID`
- `Created`
- `Modified`
- `Author`
- `Editor`
- `ContentType`
- `Attachments` za liste
- `FileLeafRef` za biblioteke

Ova polja ne treba brisati niti koristiti kao zamenu za poslovne identifikatore.

Posebno:

- `ID` nije poslovni delovodni broj
- `Created` nije uvek isto što i datum zavođenja
- `Author` nije uvek isto što i korisnik koji je poslovno odgovoran za zavođenje
- `FileLeafRef` nije isto što i naslov ili delovodni broj dokumenta

---

## 10. Indeksiranje i unique constraints

### 10.1 Potvrđeno stanje

Na osnovu dostavljenog XML-a, za ključna polja je vidljivo:

| Polje | Lista/biblioteka | Indexed | EnforceUniqueValues |
|---|---|---|---|
| `DelovodniBroj` | `Shared Documents` | false | false |
| `RezervisaniBroj` | `RezervisaniBrojevi` | false | false |
| `Title` | `AppConfig` | false | false |
| `Title` | `RezervisaniBrojevi` | false | false |

### 10.2 Rizik

Za sistem elektronske pisarnice ovo je ozbiljan rizik jer ključna poslovna polja nisu potvrđena kao indeksirana ili jedinstvena.

### 10.3 Preporuka

Obavezno razmotriti:

- indeks na `DelovodniBroj`
- unique values na `DelovodniBroj` ili `DelovodniBrojKey`
- indeks na `FinalDelovodniBroj`
- unique values na `FinalDelovodniBroj`
- indeks na `RequestId`
- indeks na `Status`
- indeks na `Godina`
- indeks na `DelovodnaKnjiga`

---

## 11. Predloženi target data model za numbering

Za robustan numbering servis preporučuje se odvojena lista:

```text
NumberingCounters
```

Minimalna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Title` | Single line of text | Unique key, npr. DK-2026 |
| `DelovodnaKnjiga` | Single line of text | Knjiga |
| `Godina` | Number | Godina |
| `LastNumber` | Number | Poslednji iskorišćen broj |
| `ETagShadow` | Single line of text | Opcionalno za audit |
| `LockedByRunId` | Single line of text | Ko trenutno pokušava update |
| `Modified` | Date and Time | Sistemski ETag oslonac |

Proces:

1. Flow učita counter item.
2. Izračuna sledeći broj.
3. Pokuša update sa ETag / If-Match kontrolom.
4. Ako update uspe, broj je rezervisan.
5. Ako update ne uspe, drugi proces je bio brži.
6. Flow čeka kratko i ponavlja pokušaj.
7. Nakon uspeha kreira zapis u `RezervisaniBrojevi`.
8. Finalni dokument dobija `DelovodniBroj`.

---

## 12. Predloženi target data model za audit

Za enterprise rešenje preporučuje se posebna lista:

```text
AuditLog
```

Minimalna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Title` | Single line of text | Kratak opis događaja |
| `EventType` | Choice/Text | Tip događaja |
| `EntityType` | Choice/Text | Document/Numbering/Email/Export |
| `EntityId` | Single line of text | ID entiteta |
| `CorrelationId` | Single line of text | Trace ID |
| `FlowRunId` | Single line of text | Power Automate run |
| `UserEmail` | Single line of text | Korisnik |
| `Status` | Choice | Success/Error/Warning |
| `Message` | Multiple lines of text | Detalji |
| `Payload` | Multiple lines of text | JSON payload, ako je potrebno |
| `Created` | Date and Time | Sistemsko vreme |

---

## 13. Power Apps implikacije

Canvas aplikacija treba da koristi SharePoint data model na sledeći način:

Dozvoljeno:

- učitavanje konfiguracije iz `AppConfig`
- prikaz dokumenata iz biblioteka
- prikaz statusa
- slanje zahteva flow-u
- prikaz vraćenog delovodnog broja

Nije preporučeno:

- direktno računanje sledećeg delovodnog broja
- direktan upis finalnog delovodnog broja bez flow-a
- oslanjanje na lokalne kolekcije za jedinstvenost
- masovno učitavanje velikih lista bez delegacije
- klijentsko filtriranje preko neindeksiranih kolona

---

## 14. Power Automate implikacije

Power Automate treba da bude centralni sloj za:

- validaciju ulaznih podataka
- generisanje broja
- rezervaciju broja
- finalni upis dokumenta
- rollback ili cancel rezervacije
- audit
- error handling
- retry
- kontrolisani response prema Power Apps-u

Power Automate ne sme da koristi naivan obrazac:

```text
Get items -> max broj + 1 -> create item
```

bez ETag / unique / retry kontrole.

Takav obrazac nije bezbedan kod paralelnog rada više korisnika.

---

## 15. Ključni zaključci

### Potvrđeno

- SharePoint je centralni data/storage layer.
- Postoji `AppConfig` lista.
- Postoji `RezervisaniBrojevi` lista.
- Postoji `Shared Documents` biblioteka.
- `Shared Documents` ima polje `DelovodniBroj`.
- Postoji `EmailDocuments` biblioteka.
- Postoji `Exports` biblioteka.

### Kritično

- `DelovodniBroj` nije potvrđen kao unique.
- `DelovodniBroj` nije potvrđen kao indexed.
- `RezervisaniBroj` nije potvrđen kao unique.
- Nije potvrđena concurrency-safe logika.
- Nije potvrđen audit model.
- Nije potvrđena veza između rezervacije broja i finalnog dokumenta.

### Preporuka

Za novu enterprise verziju obavezno projektovati:

- centralizovan numbering servis
- atomic/optimistic locking
- retry logiku
- unique constraint na finalnom ključu
- audit log
- statusni model za rezervacije
- jasnu vezu između rezervacije i dokumenta
- indeksirana ključna polja

---

## 16. Otvorena pitanja

1. Da li `AppConfig.Config` sadrži JSON?
2. Koji su konkretni zapisi u `AppConfig` listi?
3. Da li `RezervisaniBrojevi` trenutno koristi neki flow?
4. Da li `RezervisaniBroj` može da se ponovi po godini ili knjizi?
5. Koji je format finalnog delovodnog broja?
6. Da li `DelovodniBroj` u `Shared Documents` mora biti globalno jedinstven?
7. Da li postoji posebna lista predmeta osim biblioteke dokumenata?
8. Da li postoji audit lista?
9. Da li korisnici imaju direktan write access na SharePoint?
10. Da li finalni upis uvek ide preko Power Automate-a?
11. Da li postoje folderi u bibliotekama?
12. Da li se email dokumenti automatski zavode ili samo čuvaju?
13. Ko pokreće export proces?
14. Da li postoji retention policy za export fajlove?

---

## 17. Sledeći dokument

Sledeći dokument:

```text
04-appconfig-analysis.md
```

Taj dokument treba detaljno da opiše ulogu `AppConfig` liste, preporučeni JSON model, rizike i način dokumentovanja konfiguracija u GitHub repozitorijumu.
