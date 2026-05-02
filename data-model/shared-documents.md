# Data Model — Shared Documents

## 1. Svrha dokumenta

Ovaj dokument opisuje SharePoint document library:

```text
Shared Documents
```

U rešenju **DocCentral v6.0**, ova biblioteka predstavlja glavnu lokaciju za čuvanje dokumenata koji su zavedeni ili povezani sa procesom zavođenja.

Posebno važna polja u ovoj biblioteci su:

```text
DelovodniBroj
EdokumentID
Edokument
Attachment
```

Ova biblioteka je deo centralnog Document Central procesa i povezana je sa Power Apps aplikacijom i Power Automate flow-ovima.

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Postojanje biblioteke `Shared Documents` | POTVRĐENO |
| SharePoint scope biblioteke | POTVRĐENO |
| Ključna metadata polja | POTVRĐENO |
| Polje `DelovodniBroj` | POTVRĐENO |
| Polje `EdokumentID` | POTVRĐENO |
| Polje `Edokument` | POTVRĐENO |
| Polje `Attachment` | POTVRĐENO |
| Upload dokumenta preko Power Automate flow-a `Upload Doc` | POTVRĐENO |
| Tačna interna logika flow-a `Upload Doc` | NEPOZNATO |
| Da li je `DelovodniBroj` unique u biblioteci | NEPOZNATO |
| Da li je biblioteka glavna storage lokacija za sve dokumente | POTVRĐENO / PRETPOSTAVKA |

---

## 3. SharePoint lokacija

Site:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

Biblioteka:

```text
/sites/DocumentCentralv6.0/Shared Documents
```

List GUID iz dostavljenog XML-a:

```text
8cc979f6-6d95-4f5d-9000-9e00229ce9ee
```

REST endpoint obrazac:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/Web/Lists(guid'8cc979f6-6d95-4f5d-9000-9e00229ce9ee')
```

---

## 4. Poslovna uloga biblioteke

`Shared Documents` je glavna biblioteka za čuvanje dokumenata.

Na osnovu dostavljenih metadata odgovora, biblioteka sadrži poslovna metadata polja koja omogućavaju povezivanje fajla sa delovodnim brojem i statusom elektronskog dokumenta.

Logička uloga:

```text
Dokument / fajl
   |
   | metadata
   v
DelovodniBroj
Edokument
EdokumentID
Attachment
```

Biblioteka najverovatnije služi za:

- čuvanje finalnih dokumenata
- povezivanje dokumenta sa delovodnim brojem
- označavanje da li je dokument elektronski
- označavanje da li fajl predstavlja attachment
- filtriranje dokumenata po delovodnom broju
- otvaranje dokumenata iz SharePoint prikaza ili iz aplikacije

---

## 5. Identifikovana polja

### 5.1 Poslovna polja

| Internal name | Display name | Tip | Required | Read only | Filterable | Napomena |
|---|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Da | Naziv fajla |
| `Title` | Title | Single line of text | Ne | Ne | Da | SharePoint naslov dokumenta |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Ne | Opis dokumenta |
| `DelovodniBroj` | DelovodniBroj | Single line of text | Ne | Ne | Da | Veza dokumenta sa delovodnim brojem |
| `EdokumentID` | EdokumentID | Single line of text | Ne | Ne | Da | ID elektronskog dokumenta |
| `Edokument` | Edokument | Yes/No | Ne | Ne | Da | Oznaka elektronskog dokumenta |
| `Attachment` | Attachment | Yes/No | Ne | Ne | Da | Oznaka da li je fajl attachment |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Da | Sistemsko / media tagging polje |
| `ContentType` | Content Type | Computed | Ne | Ne | Da | SharePoint content type |

---

## 6. Detalji ključnih polja

## 6.1 FileLeafRef

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
Naziv fajla u document library.
```

Zaključak:

```text
Ovo je obavezno SharePoint polje za fajl u biblioteci.
```

---

## 6.2 Title

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

Napomena:

```text
U ovoj biblioteci Title je sealed SharePoint polje.
```

XML pokazuje:

```text
ShowInNewForm="FALSE"
ShowInFileDlg="FALSE"
Sealed="TRUE"
```

Zaključak:

```text
Title najverovatnije nije centralno poslovno polje za dokument, već sistemsko / pomoćno metadata polje.
```

---

## 6.3 DelovodniBroj

SharePoint metadata:

```text
InternalName: DelovodniBroj
DisplayName: DelovodniBroj
TypeAsString: Text
FieldTypeKind: 2
Required: false
ReadOnlyField: false
Filterable: true
MaxLength: 255
EnforceUniqueValues: false
Indexed: false
```

Namena:

```text
Čuva delovodni broj povezan sa dokumentom.
```

Zaključak:

```text
Ovo je jedno od najvažnijih poslovnih polja u biblioteci.
```

Rizik:

```text
U dostavljenom XML-u nije potvrđeno da je `DelovodniBroj` unique.
```

Važna enterprise napomena:

```text
Ako se isti delovodni broj može pojaviti na više fajlova zbog priloga, onda `DelovodniBroj` u biblioteci ne može uvek biti unique.
```

Mogući scenariji:

| Scenario | Da li `DelovodniBroj` može biti unique u biblioteci? |
|---|---|
| Jedan dokument = jedan fajl | Može |
| Jedan predmet = više fajlova / priloga | Ne mora |
| Glavni dokument + attachment-i dele isti broj | Ne |
| Delovodni broj je unique samo u glavnoj listi predmeta | Da, ali ne u biblioteci |

Preporuka:

```text
Finalni unique constraint za delovodni broj treba držati na nivou glavnog predmeta / registry zapisa, ne nužno na svim fajlovima u biblioteci.
```

---

## 6.4 EdokumentID

SharePoint metadata:

```text
InternalName: EdokumentID
DisplayName: EdokumentID
TypeAsString: Text
FieldTypeKind: 2
Required: false
ReadOnlyField: false
Filterable: true
MaxLength: 255
EnforceUniqueValues: false
Indexed: false
```

Namena:

```text
Identifikator elektronskog dokumenta.
```

Moguća upotreba:

- povezivanje sa eksternim e-dokument sistemom
- povezivanje sa SEF / e-faktura / elektronskom dostavom
- identifikacija dokumenta koji nije samo skenirani fajl
- veza sa email ili integracionim procesom

Status:

```text
POTVRĐENO: polje postoji.
NEPOZNATO: tačan izvor i format vrednosti nije dostavljen.
```

---

## 6.5 Edokument

SharePoint metadata:

```text
InternalName: Edokument
DisplayName: Edokument
TypeAsString: Boolean
FieldTypeKind: 8
Required: false
ReadOnlyField: false
Filterable: true
DefaultValue: 0
```

Namena:

```text
Označava da li dokument jeste elektronski dokument.
```

Vrednosti:

| SharePoint vrednost | Značenje |
|---|---|
| `0` | Nije elektronski dokument |
| `1` | Elektronski dokument |

Power Automate filter primer:

```text
Edokument eq 1
```

ili:

```text
Edokument eq 0
```

Power Automate expression primer:

```text
equals(triggerBody()?['Edokument'], true)
```

---

## 6.6 Attachment

SharePoint metadata:

```text
InternalName: Attachment
DisplayName: Attachment
TypeAsString: Boolean
FieldTypeKind: 8
Required: false
ReadOnlyField: false
Filterable: true
DefaultValue: 0
```

Namena:

```text
Označava da li fajl predstavlja attachment / prilog.
```

Moguća poslovna pravila:

- glavni dokument ima `Attachment = false`
- prilozi imaju `Attachment = true`
- attachment-i mogu naslediti isti `DelovodniBroj`
- attachment-i mogu biti povezani sa glavnim dokumentom preko dodatnog ID-a

Status:

```text
POTVRĐENO: polje postoji.
PRETPOSTAVKA: poslovna pravila nisu dostavljena.
```

---

## 6.7 _ExtendedDescription

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
Opis dokumenta.
```

Napomena:

```text
Polje je rich text i nije filterable.
```

Preporuka:

```text
Ne koristiti ovo polje za tehničke filtere ili ključnu poslovnu logiku.
```

---

## 6.8 MediaServiceImageTags

SharePoint metadata:

```text
InternalName: MediaServiceImageTags
DisplayName: Image Tags
TypeAsString: TaxonomyFieldTypeMulti
TypeDisplayName: Managed Metadata
AllowMultipleValues: true
Group: _Hidden
Description: Field created by MediaTA
```

Zaključak:

```text
Ovo je SharePoint / Microsoft 365 sistemsko polje za media/image tagging.
```

Preporuka:

```text
Ne koristiti ga kao deo poslovne logike DocCentral aplikacije.
```

---

## 7. Upload dokumenta

Korisnik je potvrdio:

```text
Upload dokumenta ide preko Power Automate flow-a Upload Doc.
```

To znači:

```text
Canvas App ne upisuje fajl direktno u SharePoint biblioteku.
```

Logički tok:

```text
Canvas App
   |
   | šalje fajl / metadata
   v
Power Automate flow
Upload Doc
   |
   | kreira fajl u biblioteci
   | upisuje metadata
   v
Shared Documents
```

Status:

```text
POTVRĐENO: upload ide preko flow-a `Upload Doc`.
NEPOZNATO: interna implementacija flow-a nije dostavljena.
```

---

## 8. Veza sa zavođenjem dokumenta

Biblioteka `Shared Documents` je povezana sa procesom zavođenja kroz polje:

```text
DelovodniBroj
```

Moguća veza:

```text
DelovodniBroj u glavnom predmetu
   |
   v
DelovodniBroj na fajlovima u Shared Documents
```

Primer filter link obrasca koji se koristi u Document Central scenarijima:

```text
/Dokumenta/Forms/AllItems.aspx?FilterField1=DelovodniBroj&FilterValue1=<encoded value>&FilterType1=Text
```

Napomena:

```text
Tačan URL za ovu konkretnu biblioteku zavisi od view-a i naziva biblioteke.
```

---

## 9. Odnos prema delovodnom broju

Postoje dva odvojena pojma:

| Pojam | Lokacija | Namena |
|---|---|---|
| Rezervacija broja | `RezervisaniBrojevi` | Kontrola i dodela broja |
| Metadata na dokumentu | `Shared Documents.DelovodniBroj` | Povezivanje fajla sa brojem |

Važno:

```text
Biblioteka ne treba da bude jedini izvor istine za generisanje delovodnog broja.
```

Izvor istine za dodelu broja treba da bude:

```text
Power Automate flow + RezervisaniBrojevi / counter mehanizam + unique constraint
```

---

## 10. Rizik duplog delovodnog broja

Ako više fajlova može imati isti `DelovodniBroj`, biblioteka sama po sebi ne može garantovati jedinstvenost predmeta.

Primer:

```text
Glavni dokument: Os.Del.Br. 100/2026
Prilog 1: Os.Del.Br. 100/2026
Prilog 2: Os.Del.Br. 100/2026
```

U tom scenariju duplikat u biblioteci nije greška.

Zato se mora razlikovati:

```text
Duplikat fajla sa istim delovodnim brojem
```

od:

```text
Duplikat dva različita predmeta sa istim delovodnim brojem
```

Preporuka:

```text
Unique constraint za finalni poslovni broj držati na nivou registry/predmet entiteta, a u document library dozvoliti više fajlova po istom broju ako su to prilozi.
```

---

## 11. Preporučeni metadata model za novu verziju

Postojeća polja su minimalna.

Za enterprise verziju preporučuje se proširenje metadata modela.

Predložena dodatna polja:

| Internal name | Tip | Namena |
|---|---|---|
| `RegistryItemId` | Number/Text | ID glavnog predmeta |
| `RegistryNumber` | Text | Finalni delovodni broj |
| `RegistryYear` | Number | Godina |
| `RegistryPrefix` | Text | Prefix broja |
| `DocumentRole` | Choice | Main / Attachment / EmailAttachment / Export |
| `SourceType` | Choice | ManualUpload / Email / Import / Generated |
| `SourceFlowRunId` | Text | Correlation ID Power Automate run-a |
| `UploadedByEmail` | Text | Ko je pokrenuo upload |
| `UploadedAt` | DateTime | Vreme upload-a |
| `OriginalFileName` | Text | Originalni naziv fajla |
| `DocumentStatus` | Choice | Uploaded / Registered / Archived / Failed |
| `Checksum` | Text | Hash fajla za detekciju duplikata |
| `ErrorMessage` | Multiple lines | Greška ako upload nije uspeo |

---

## 12. Preporučeni DocumentRole model

| Vrednost | Opis |
|---|---|
| `Main` | Glavni dokument |
| `Attachment` | Prilog glavnom dokumentu |
| `EmailAttachment` | Prilog iz email intake procesa |
| `GeneratedPdf` | Sistemski generisan PDF |
| `Export` | Export fajl |
| `Other` | Ostalo |

---

## 13. Preporučeni SourceType model

| Vrednost | Opis |
|---|---|
| `ManualUpload` | Korisnik ručno dodao dokument kroz aplikaciju |
| `PowerApps` | Dokument poslat iz Canvas aplikacije |
| `Email` | Dokument došao preko shared mailbox procesa |
| `SAP` | Dokument došao iz SAP integracije |
| `SEF` | Dokument došao iz SEF/e-faktura procesa |
| `SystemGenerated` | Dokument generisan od sistema |
| `Export` | Export iz sistema |

---

## 14. Permissions i security napomena

U prethodno definisanoj arhitekturi za DocCentral važi pravilo:

```text
Korisnici imaju Read Only prava nad SharePoint-om, a upise izvršava Power Automate.
```

Za `Shared Documents` to znači:

```text
Korisnik ne treba direktno da kreira / menja fajlove u biblioteci.
```

Preporučeni model:

```text
Power Apps
   |
   | payload
   v
Power Automate service account
   |
   | create file / update metadata
   v
Shared Documents
```

Prednosti:

- centralna kontrola upisa
- manji rizik od pogrešnog metadata
- lakša kontrola dozvola
- bolji audit
- konzistentniji proces

Rizik:

```text
Service account mora imati pravilno definisana prava i licencu.
```

---

## 15. Indexing preporuke

U dostavljenom XML-u ključna custom polja nisu potvrđena kao indeksirana.

Preporučena indeksirana polja:

| Polje | Razlog |
|---|---|
| `DelovodniBroj` | Pretraga dokumenata po broju |
| `Edokument` | Filtriranje elektronskih dokumenata |
| `Attachment` | Razdvajanje glavnih dokumenata i priloga |
| `EdokumentID` | Povezivanje sa spoljnim sistemom |
| `Created` | Sortiranje i audit |
| `Modified` | Kontrola izmena |

Ako se uvedu dodatna polja:

| Polje | Razlog |
|---|---|
| `RegistryItemId` | Povezivanje sa predmetom |
| `DocumentRole` | Razdvajanje glavnog dokumenta i priloga |
| `SourceType` | Analitika i filteri |
| `DocumentStatus` | Operativni nadzor |

---

## 16. Power Automate filter primeri

### 16.1 Dokumenti po delovodnom broju

```text
DelovodniBroj eq 'Os.Del.Br. 100/2026'
```

### 16.2 Samo elektronski dokumenti

```text
Edokument eq 1
```

### 16.3 Samo dokumenti koji nisu attachment

```text
Attachment eq 0
```

### 16.4 Attachment-i za konkretan delovodni broj

```text
DelovodniBroj eq 'Os.Del.Br. 100/2026' and Attachment eq 1
```

---

## 17. Power Apps filter primeri

Ako se koristi kolekcija dokumenata:

```powerfx
Filter(
    colDocuments,
    DelovodniBroj = varSelectedDelovodniBroj
)
```

Samo glavni dokumenti:

```powerfx
Filter(
    colDocuments,
    DelovodniBroj = varSelectedDelovodniBroj &&
    Attachment = false
)
```

Samo prilozi:

```powerfx
Filter(
    colDocuments,
    DelovodniBroj = varSelectedDelovodniBroj &&
    Attachment = true
)
```

---

## 18. Known issues

| Problem | Uticaj | Preporuka |
|---|---|---|
| `DelovodniBroj` nije potvrđeno indeksiran | Spor filter na velikoj biblioteci | Indeksirati polje |
| `DelovodniBroj` nije potvrđeno unique | Ne može garantovati jedinstvenost | Unique držati na registry entitetu |
| `Attachment` je boolean bez dodatne veze | Teško povezivanje priloga sa glavnim dokumentom | Dodati `ParentDocumentId` ili `RegistryItemId` |
| Nema potvrđenog `DocumentStatus` polja | Teško praćenje lifecycle-a | Dodati status |
| Nema potvrđenog `SourceFlowRunId` | Teži audit | Dodati correlation ID |
| Nema potvrđenog checksum-a | Teža detekcija duplikata fajlova | Dodati hash/checksum |
| Upload flow nije analiziran | Nepoznata logika i greške | Dostaviti export flow-a `Upload Doc` |

---

## 19. Minimalni acceptance kriterijumi za novu verziju

```text
AC-SD-001:
Upload dokumenta mora ići kroz kontrolisan backend proces, ne direktno iz korisničkog SharePoint pristupa.

AC-SD-002:
Svaki dokument mora imati jasno povezivanje sa predmetom ili delovodnim brojem.

AC-SD-003:
Glavni dokument i attachment-i moraju biti jasno razdvojeni.

AC-SD-004:
Biblioteka mora podržati pretragu po delovodnom broju.

AC-SD-005:
Upload flow mora vratiti standardizovan success/error response.

AC-SD-006:
Svaki upload mora imati audit / correlation ID.

AC-SD-007:
Greška u upload-u ne sme ostaviti nekonzistentan predmet bez jasnog statusa.

AC-SD-008:
Ako se koristi service account, korisnički identitet inicijatora mora biti upisan u metadata ili audit.
```

---

## 20. Test scenariji

### Test 1 — upload glavnog dokumenta

```text
1. Korisnik pokrene zavođenje.
2. Flow `Upload Doc` kreira fajl u Shared Documents.
3. Metadata se upisuje na fajl.
4. Dokument dobija DelovodniBroj.
```

Očekivanje:

```text
Fajl postoji u biblioteci i metadata su popunjena.
```

---

### Test 2 — upload attachment-a

```text
1. Korisnik dodaje prilog.
2. Flow kreira fajl.
3. Attachment = true.
4. DelovodniBroj je isti kao kod glavnog dokumenta.
```

Očekivanje:

```text
Prilog je jasno označen kao attachment.
```

---

### Test 3 — filter po delovodnom broju

```text
1. Korisnik otvara predmet.
2. Sistem filtrira dokumente po DelovodniBroj.
```

Očekivanje:

```text
Prikazani su samo dokumenti povezani sa tim brojem.
```

---

### Test 4 — email dokument ne sme završiti kao pogrešan tip

```text
1. Dokument stiže kroz email proces.
2. Ako ide u EmailDocuments, ne sme pogrešno završiti u Shared Documents bez pravila.
```

Očekivanje:

```text
SourceType / DocumentRole jasno razdvajaju poreklo dokumenta.
```

---

## 21. Otvorena pitanja

| Pitanje | Status |
|---|---|
| Da li `Shared Documents` čuva sve finalne dokumente ili samo ručno uploadovane? | NEPOZNATO |
| Da li attachment-i dele isti `DelovodniBroj` kao glavni dokument? | PRETPOSTAVKA |
| Da li postoji veza attachment-a sa glavnim dokumentom osim preko `DelovodniBroj`? | NEPOZNATO |
| Da li `Upload Doc` vraća JSON response u Power Apps? | NEPOZNATO |
| Da li `Upload Doc` upisuje `DelovodniBroj` odmah ili kasnije drugi flow? | NEPOZNATO |
| Da li se `EdokumentID` puni iz eksternog sistema? | NEPOZNATO |
| Koji je format `EdokumentID`? | NEPOZNATO |
| Da li se `Edokument` koristi u trigger condition-ima? | NEPOZNATO |
| Da li korisnici imaju direktan pristup biblioteci ili samo kroz aplikaciju? | PRETPOSTAVKA |
| Da li postoji versioning na biblioteci? | NEPOZNATO |
| Da li postoje retention label pravila? | NEPOZNATO |
| Da li postoji finalni arhivski PDF proces? | NEPOZNATO |

---

## 22. Zaključak

`Shared Documents` je glavna document library komponenta u DocCentral v6.0 rešenju.

Potvrđena su ključna polja:

```text
DelovodniBroj
EdokumentID
Edokument
Attachment
```

Potvrđeno je da upload dokumenata ide preko Power Automate flow-a:

```text
Upload Doc
```

Najvažnija arhitektonska napomena:

```text
Biblioteka čuva dokumente i metadata, ali ne treba da bude jedini izvor istine za generisanje i garanciju jedinstvenog delovodnog broja.
```

Za enterprise verziju potrebno je:

- jasno razdvojiti glavni dokument i attachment-e
- dodati correlation ID
- dodati status dokumenta
- indeksirati ključna polja
- obezbediti audit upload procesa
- povezati dokumente sa glavnim registry entitetom
- zadržati generisanje delovodnog broja u backend logici

---

## 23. Povezani dokumenti

```text
README.md
docs/03-sharepoint-data-model.md
docs/05-document-libraries.md
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
docs/10-known-issues-technical-debt.md
data-model/appconfig.md
data-model/rezervisani-brojevi.md
data-model/email-documents.md
data-model/exports.md
architecture/current-architecture.md
architecture/numbering-sequence.md
backlog/known-issues.md
backlog/open-questions.md
```
