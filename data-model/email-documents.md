# Data Model — EmailDocuments

## 1. Svrha dokumenta

Ovaj dokument opisuje SharePoint document library:

```text
EmailDocuments
```

U rešenju **DocCentral v6.0**, ova biblioteka predstavlja posebnu lokaciju za dokumente koji dolaze kroz email proces.

Korisnik je potvrdio:

```text
EmailDocuments je posebna funkcionalnost gde Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.
```

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Postojanje biblioteke `EmailDocuments` | POTVRĐENO |
| SharePoint scope biblioteke | POTVRĐENO |
| Veza sa shared mailbox procesom | POTVRĐENO OD KORISNIKA |
| Power Automate čita shared mailbox | POTVRĐENO OD KORISNIKA |
| Proces radi sa mailovima koji imaju attachment | POTVRĐENO OD KORISNIKA |
| Postojanje polja `PosiljalacEmail` | POTVRĐENO |
| Postojanje polja `Posiljalac` | POTVRĐENO |
| Tačan naziv flow-a za email intake | NEPOZNATO |
| Tačna trigger logika flow-a | NEPOZNATO |
| Da li email dokument dobija delovodni broj odmah | NEPOZNATO |
| Da li email dokument kasnije prelazi u `Shared Documents` | NEPOZNATO |

---

## 3. SharePoint lokacija

Site:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

Biblioteka:

```text
/sites/DocumentCentralv6.0/EmailDocuments
```

List GUID iz dostavljenog XML-a:

```text
574bcbe1-9cbb-419a-b348-a1ab2aa0bc76
```

REST endpoint obrazac:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/Web/Lists(guid'574bcbe1-9cbb-419a-b348-a1ab2aa0bc76')
```

---

## 4. Poslovna uloga biblioteke

`EmailDocuments` služi kao document library za dokumente pristigle emailom.

Potvrđena poslovna funkcionalnost:

```text
Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.
```

Logički tok:

```text
Shared mailbox
   |
   | novi email sa attachment-om
   v
Power Automate flow
   |
   | čitanje maila
   | ekstrakcija attachment-a
   | upis fajla / metadata
   v
EmailDocuments
```

Biblioteka može imati jednu ili više uloga:

1. Privremeni intake storage za dokumente iz emaila.
2. Finalna biblioteka za email dokumente.
3. Staging zona pre zavođenja.
4. Audit zona za primljene mailove i attachment-e.

Status ovih uloga:

```text
POTVRĐENO: koristi se za email dokumente.
NEPOZNATO: da li je staging ili finalna lokacija.
```

---

## 5. Identifikovana polja

| Internal name | Display name | Tip | Required | Read only | Filterable | Napomena |
|---|---|---|---|---|---|---|
| `FileLeafRef` | Name | File | Da | Ne | Da | Naziv fajla |
| `Title` | Title | Single line of text | Ne | Ne | Da | SharePoint naslov |
| `_ExtendedDescription` | Description | Multiple lines of text | Ne | Ne | Ne | Opis dokumenta |
| `MediaServiceImageTags` | Image Tags | Managed Metadata | Ne | Ne | Da | Sistemsko media tagging polje |
| `PosiljalacEmail` | PosiljalacEmail | Single line of text | Ne | Ne | Da | Email adresa pošiljaoca |
| `Posiljalac` | Posiljalac | Single line of text | Ne | Ne | Da | Naziv / ime pošiljaoca |
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
Naziv attachment fajla koji je sačuvan u biblioteci.
```

U email intake procesu ovo najverovatnije odgovara originalnom nazivu attachment-a.

Status:

```text
POTVRĐENO: polje postoji.
PRETPOSTAVKA: vrednost dolazi iz attachment filename-a.
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

XML pokazuje:

```text
ShowInNewForm="FALSE"
ShowInFileDlg="FALSE"
Sealed="TRUE"
```

Zaključak:

```text
Title nije primarno poslovno polje za email dokument.
```

Preporuka:

```text
Za email procese koristiti jasna custom metadata polja, ne oslanjati se na Title.
```

---

## 6.3 PosiljalacEmail

SharePoint metadata:

```text
InternalName: PosiljalacEmail
DisplayName: PosiljalacEmail
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
Čuva email adresu pošiljaoca.
```

Primer vrednosti:

```text
partner@example.com
```

Moguća upotreba:

- pretraga dokumenata po pošiljaocu
- povezivanje pošiljaoca sa partnerom
- audit email intake procesa
- automatska klasifikacija dokumenata
- pravila za dozvoljene ili blokirane pošiljaoce

Preporuka:

```text
Indeksirati `PosiljalacEmail` ako se koristi za filtere i pretragu.
```

---

## 6.4 Posiljalac

SharePoint metadata:

```text
InternalName: Posiljalac
DisplayName: Posiljalac
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
Čuva ime, naziv ili prikazani naziv pošiljaoca.
```

Primeri mogućih vrednosti:

```text
Petar Petrović
Company Name d.o.o.
Accounts Payable
```

Preporuka:

```text
Koristiti `PosiljalacEmail` kao stabilniji identifikator, a `Posiljalac` kao display podatak.
```

---

## 6.5 _ExtendedDescription

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

Moguća email upotreba:

- subject emaila
- deo body-ja emaila
- opis attachment-a
- tehnička napomena

Status:

```text
NEPOZNATO: da li se ovo polje puni iz email subject-a ili body-ja.
```

Preporuka:

```text
Ako se koristi email subject ili body, bolje je dodati posebna polja `EmailSubject` i `EmailBodyPreview`.
```

---

## 6.6 MediaServiceImageTags

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
Ovo je sistemsko SharePoint / Microsoft 365 polje.
```

Preporuka:

```text
Ne koristiti ovo polje za poslovnu logiku email intake procesa.
```

---

## 7. Email intake proces

Potvrđeni opis:

```text
Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.
```

Minimalna očekivana logika flow-a:

```text
1. Trigger na novi email u shared mailbox-u.
2. Provera da email ima attachment.
3. Iteracija kroz attachment-e.
4. Kreiranje fajla u EmailDocuments.
5. Upis metadata:
   - PosiljalacEmail
   - Posiljalac
   - naziv fajla
   - eventualno subject/body
6. Poziv procesa za zavođenje ili priprema za zavođenje.
```

Status:

```text
POTVRĐENO: funkcionalnost postoji.
NEPOZNATO: tačna implementacija nije dostavljena.
```

---

## 8. Predložena Power Automate arhitektura za email intake

Preporučeni tok:

```text
Shared Mailbox Trigger
   |
   v
Validate Email
   |
   | has attachment?
   v
Extract Attachments
   |
   v
Create Files in EmailDocuments
   |
   v
Create / reserve registry number
   |
   v
Update metadata
   |
   v
Audit log
```

Poželjno je da flow bude podeljen na logičke Scope blokove:

```text
Scope - Initialize
Scope - Validate Email
Scope - Process Attachments
Scope - Register Document
Scope - Update Metadata
Scope - Audit Success
Scope - Error Handler
```

---

## 9. Rush condition / race condition napomena

Korisnik je posebno naglasio da postoji poseban flow zbog race condition problema:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Ovaj flow je vezan za dodelu delovodnog broja i treba ga tretirati kao centralni concurrency-safe mehanizam.

Za email dokumente važi isto pravilo:

```text
Email intake proces ne sme samostalno i nekontrolisano generisati delovodni broj.
```

Ako email flow zavodi dokument, mora koristiti isti centralni mehanizam za broj.

Preporuka:

```text
Svi kanali zavođenja — Power Apps, email, import, SAP, SEF — moraju koristiti isti backend servis za dodelu delovodnog broja.
```

---

## 10. Veza sa delovodnim brojem

U dostavljenom XML-u za `EmailDocuments` nisu potvrđena polja:

```text
DelovodniBroj
EdokumentID
Edokument
Attachment
```

Za razliku od `Shared Documents`, ova biblioteka ima email-specifična polja:

```text
PosiljalacEmail
Posiljalac
```

Zaključak:

```text
Na osnovu dostavljenog XML-a nije potvrđeno da `EmailDocuments` direktno čuva `DelovodniBroj`.
```

Moguća arhitektura:

```text
EmailDocuments
   |
   | intake / staging
   v
Power Automate registration process
   |
   v
Shared Documents / Registry list
```

Alternativno:

```text
EmailDocuments
   |
   | final email document library
   v
Delovodni broj se čuva u drugoj listi / predmetu
```

Status:

```text
NEPOZNATO: gde se tačno čuva veza između email dokumenta i delovodnog broja.
```

---

## 11. Preporučeni metadata model za novu verziju

Za enterprise email intake proces preporučuju se dodatna polja.

| Internal name | Tip | Namena |
|---|---|---|
| `EmailMessageId` | Text | Jedinstveni ID email poruke |
| `EmailConversationId` | Text | ID conversation/thread-a |
| `EmailSubject` | Text | Subject emaila |
| `EmailBodyPreview` | Multiple lines | Skraćeni body |
| `ReceivedDateTime` | DateTime | Kada je email primljen |
| `MailboxAddress` | Text | Shared mailbox adresa |
| `AttachmentName` | Text | Originalni naziv attachment-a |
| `AttachmentContentType` | Text | MIME tip attachment-a |
| `AttachmentSize` | Number | Veličina attachment-a |
| `AttachmentIndex` | Number | Redni broj attachment-a u emailu |
| `ProcessingStatus` | Choice | New / Processing / Registered / Failed / Ignored |
| `ProcessingError` | Multiple lines | Opis greške |
| `FlowRunId` | Text | Power Automate run ID |
| `RegistryItemId` | Text/Number | Veza sa predmetom |
| `DelovodniBroj` | Text | Delovodni broj, ako se čuva u ovoj biblioteci |
| `SourceType` | Choice | Email |
| `ProcessedAt` | DateTime | Kada je proces završen |

---

## 12. Preporučeni status model

| Status | Opis |
|---|---|
| `New` | Email dokument je sačuvan, ali nije obrađen |
| `Processing` | Flow trenutno obrađuje dokument |
| `Registered` | Dokument je uspešno zaveden |
| `Ignored` | Dokument je namerno preskočen |
| `Failed` | Došlo je do greške |
| `Duplicate` | Detektovan duplikat |
| `NeedsManualReview` | Potrebna ručna provera |

---

## 13. Preporučena pravila za attachment-e

Email može imati više attachment-a.

Zato jedan email može proizvesti više fajlova:

```text
Email 1
   |
   ├── Attachment 1
   ├── Attachment 2
   └── Attachment 3
```

Preporučena pravila:

1. Svaki attachment mora imati poseban zapis/fajl.
2. Svi attachment-i iz istog emaila treba da dele `EmailMessageId`.
3. Svaki attachment treba da ima `AttachmentIndex`.
4. Ako se svi attachment-i zavode kao jedan predmet, treba da dele isti `RegistryItemId`.
5. Ako se svaki attachment zavodi kao poseban predmet, svaki mora proći kroz centralni numbering flow.
6. Greška na jednom attachment-u ne sme sakriti status ostalih attachment-a.

---

## 14. Preporučena pravila za filtriranje mailova

Flow treba da filtrira emailove pre obrade.

Minimalne provere:

| Provera | Razlog |
|---|---|
| Email ima attachment | Da se ne obrađuju prazni mailovi |
| Attachment nije inline slika | Da se ne zavode potpisi i logotipi |
| Attachment ima dozvoljeni extension | Bezbednost |
| Attachment nije prevelik | Stabilnost flow-a |
| Pošiljalac nije blokiran | Bezbednost |
| Email nije već obrađen | Sprečavanje duplikata |

---

## 15. Detekcija duplikata

Email proces mora imati zaštitu od duple obrade.

Mogući uzroci duplikata:

- flow retry
- isti email pročitan više puta
- attachment poslat više puta
- korisnik ručno pokrene reprocess
- email forward kreira sličan attachment
- throttling / timeout ponovi akciju

Preporučeni ključevi za duplikate:

```text
EmailMessageId + AttachmentName + AttachmentSize
```

Bolji model:

```text
EmailMessageId + AttachmentIndex + AttachmentHash
```

Najbolji model:

```text
EmailMessageId + AttachmentId + FileChecksum
```

---

## 16. Security napomena

Email intake je visokorizičan kanal jer dokumenti dolaze spolja.

Obavezne kontrole:

- validacija extension-a
- ograničenje veličine fajla
- malware scanning preko Microsoft 365 zaštite
- blokiranje izvršnih fajlova
- audit pošiljaoca
- logovanje originalnog message ID-a
- kontrola shared mailbox pristupa
- service account za flow
- minimalna prava nad SharePoint bibliotekom

---

## 17. Permissions model

Preporučeni model:

```text
Shared mailbox
   |
   | Power Automate connection / service account
   v
Power Automate flow
   |
   | create file / update metadata
   v
EmailDocuments
```

Korisnici ne bi trebalo direktno da upisuju fajlove u `EmailDocuments`.

Preporučeno:

| Akter | Prava |
|---|---|
| Obični korisnik | Read ili bez direktnog pristupa |
| Power Automate service account | Contribute/Edit |
| Admin | Full Control |
| Audit/Compliance korisnik | Read |
| App korisnik | Pristup kroz aplikaciju, ne direktan edit |

---

## 18. Indexing preporuke

Preporučena indeksirana polja:

| Polje | Razlog |
|---|---|
| `PosiljalacEmail` | Pretraga po pošiljaocu |
| `Posiljalac` | Operativni filter |
| `Created` | Sortiranje po prijemu |
| `Modified` | Praćenje izmena |
| `ProcessingStatus` | Operativni dashboard |
| `EmailMessageId` | Duplikati i audit |
| `FlowRunId` | Troubleshooting |
| `RegistryItemId` | Veza sa predmetom |
| `DelovodniBroj` | Pretraga po broju, ako se uvede |

---

## 19. Power Automate filter primeri

### 19.1 Dokumenti po pošiljaocu

```text
PosiljalacEmail eq 'partner@example.com'
```

### 19.2 Dokumenti po imenu pošiljaoca

```text
Posiljalac eq 'Company Name d.o.o.'
```

### 19.3 Dokumenti kreirani posle određenog datuma

```text
Created ge datetime'2026-01-01T00:00:00Z'
```

### 19.4 Ako se uvede ProcessingStatus

```text
ProcessingStatus eq 'Failed'
```

---

## 20. Preporučeni Power Automate error handling

Email flow mora imati kontrolisan error handling.

Preporučeni obrazac:

```text
Scope - Try
Scope - Catch
Scope - Finally
```

U `Catch` delu:

```text
1. Upisati ProcessingStatus = Failed.
2. Upisati ProcessingError.
3. Upisati FlowRunId.
4. Ne brisati originalni fajl bez traga.
5. Poslati admin notifikaciju ako je greška kritična.
```

---

## 21. Standardizovani response / log model

Za svaki obrađeni email attachment poželjno je zapisati:

```json
{
  "emailMessageId": "",
  "attachmentName": "",
  "fileUrl": "",
  "processingStatus": "",
  "registryItemId": "",
  "delovodniBroj": "",
  "flowRunId": "",
  "error": ""
}
```

---

## 22. Test scenariji

### Test 1 — email bez attachment-a

```text
1. Email stiže u shared mailbox.
2. Email nema attachment.
3. Flow ga preskače.
```

Očekivanje:

```text
Ne kreira se fajl u EmailDocuments.
```

---

### Test 2 — email sa jednim attachment-om

```text
1. Email stiže sa jednim PDF attachment-om.
2. Flow kreira fajl u EmailDocuments.
3. Upisuje PosiljalacEmail i Posiljalac.
```

Očekivanje:

```text
Fajl postoji i metadata su popunjena.
```

---

### Test 3 — email sa više attachment-a

```text
1. Email stiže sa tri attachment-a.
2. Flow obrađuje svaki attachment.
3. Svaki attachment dobija poseban fajl.
```

Očekivanje:

```text
Sva tri fajla su kreirana i povezana istim email identifikatorom.
```

---

### Test 4 — isti email obrađen dva puta

```text
1. Flow ponovo obradi isti email.
2. Sistem proverava EmailMessageId / AttachmentHash.
```

Očekivanje:

```text
Ne dolazi do duplog zavođenja.
```

---

### Test 5 — istovremena obrada emaila i ručnog zavođenja

```text
1. Jedan korisnik ručno zavodi dokument.
2. Email flow istovremeno zavodi dokument.
3. Oba procesa traže delovodni broj.
```

Očekivanje:

```text
Brojevi se dodeljuju kroz isti centralni concurrency-safe flow i nikada se ne dupliraju.
```

---

## 23. Known issues

| Problem | Uticaj | Preporuka |
|---|---|---|
| Nije potvrđen naziv email intake flow-a | Teže dokumentovanje zavisnosti | Dostaviti export flow-a |
| Nije potvrđeno da postoji `EmailMessageId` | Rizik duple obrade | Dodati polje |
| Nije potvrđen status obrade | Teško praćenje grešaka | Dodati `ProcessingStatus` |
| Nije potvrđen `FlowRunId` | Teži troubleshooting | Dodati audit polje |
| Nije potvrđena veza sa delovodnim brojem | Nejasna relacija sa predmetom | Dodati `RegistryItemId` ili `DelovodniBroj` |
| Nije potvrđena detekcija inline slika | Rizik zavodenja potpisa/logotipa | Dodati file filter |
| Nije potvrđena validacija ekstenzija | Bezbednosni rizik | Dodati allow-list |
| Nije potvrđena retry/idempotency logika | Rizik duplikata | Dodati idempotency key |

---

## 24. Minimalni acceptance kriterijumi za novu verziju

```text
AC-EMAIL-001:
Email intake flow mora obrađivati samo emailove sa relevantnim attachment-ima.

AC-EMAIL-002:
Svaki attachment mora imati jasan audit trag.

AC-EMAIL-003:
Sistem mora sprečiti duplu obradu istog email attachment-a.

AC-EMAIL-004:
Email dokument koji se zavodi mora koristiti centralni numbering mehanizam.

AC-EMAIL-005:
Istovremeno zavođenje iz emaila i iz Canvas aplikacije ne sme proizvesti isti delovodni broj.

AC-EMAIL-006:
Svaka greška mora biti upisana u status/log.

AC-EMAIL-007:
Pošiljalac emaila mora biti sačuvan kao metadata.

AC-EMAIL-008:
Inline slike i nepotrebni attachment-i ne smeju automatski ulaziti u poslovni proces.
```

---

## 25. Otvorena pitanja

| Pitanje | Status |
|---|---|
| Koji je tačan naziv Power Automate flow-a za čitanje shared mailbox-a? | NEPOZNATO |
| Da li flow obrađuje jedan ili više shared mailbox-a? | NEPOZNATO |
| Da li email dokument odmah dobija delovodni broj? | NEPOZNATO |
| Da li se fajl iz `EmailDocuments` kopira/premešta u `Shared Documents`? | NEPOZNATO |
| Da li postoji polje za email subject? | NEPOZNATO |
| Da li postoji polje za email message ID? | NEPOZNATO |
| Da li se čuva originalni email body? | NEPOZNATO |
| Da li se inline slike filtriraju? | NEPOZNATO |
| Da li postoji allow-list ekstenzija? | NEPOZNATO |
| Da li postoji retry/idempotency zaštita? | NEPOZNATO |
| Da li se attachment-i zavode kao jedan predmet ili svaki posebno? | NEPOZNATO |
| Da li postoji ručni review ekran za email dokumente? | NEPOZNATO |

---

## 26. Zaključak

`EmailDocuments` je posebna document library komponenta za email intake proces u DocCentral v6.0 rešenju.

Potvrđena je njena uloga:

```text
Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachment-om.
```

Potvrđena su ključna email metadata polja:

```text
PosiljalacEmail
Posiljalac
```

Najvažnija arhitektonska napomena:

```text
Email kanal mora koristiti isti centralni, concurrency-safe mehanizam za dodelu delovodnog broja kao i Canvas aplikacija.
```

Za enterprise verziju potrebno je:

- dodati email message ID
- dodati processing status
- dodati flow run ID
- dodati idempotency key
- dodati attachment hash/checksum
- definisati pravila za inline attachment-e
- definisati da li se email dokumenti zavode pojedinačno ili grupno
- povezati email dokumente sa registry entitetom
- obezbediti audit i error handling

---

## 27. Povezani dokumenti

```text
README.md
docs/03-sharepoint-data-model.md
docs/05-document-libraries.md
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/09-security-permissions.md
docs/10-known-issues-technical-debt.md
data-model/shared-documents.md
data-model/appconfig.md
data-model/rezervisani-brojevi.md
data-model/exports.md
architecture/current-architecture.md
architecture/numbering-sequence.md
backlog/known-issues.md
backlog/open-questions.md
```
