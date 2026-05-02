# Data Model — RezervisaniBrojevi

## 1. Svrha dokumenta

Ovaj dokument opisuje SharePoint listu:

```text
RezervisaniBrojevi
```

Lista `RezervisaniBrojevi` je ključna za proces rezervacije i kontrole delovodnih brojeva u rešenju **DocCentral v6.0**.

Najvažniji poslovni zahtev:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

Ovaj zahtev ima najviši prioritet u postojećoj analizi i u razvoju nove enterprise verzije.

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Postojanje liste `RezervisaniBrojevi` | POTVRĐENO |
| SharePoint scope liste | POTVRĐENO |
| Ključna polja liste | POTVRĐENO |
| Uloga liste u rezervaciji brojeva | POTVRĐENO / PRETPOSTAVKA |
| Postojanje posebnog flow-a za dodelu broja | POTVRĐENO |
| Naziv flow-a za dodelu broja | POTVRĐENO |
| Concurrency problem / race condition | POTVRĐENO |
| Tačna interna logika flow-a | NEPOZNATO |
| Da li je uključena unique constraint zaštita | NEPOZNATO |
| Da li postoji audit log rezervacije | NEPOZNATO |

---

## 3. SharePoint lokacija

Site:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

Lista:

```text
/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi
```

List GUID iz dostavljenog XML-a:

```text
b7df3cc2-af04-4b78-93ae-3f7efb4c3974
```

REST endpoint obrazac:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/Web/Lists(guid'b7df3cc2-af04-4b78-93ae-3f7efb4c3974')
```

---

## 4. Identifikovana polja

### 4.1 Poslovna polja

| Internal name | Display name | Tip | Required | Read only | Filterable | Napomena |
|---|---|---|---|---|---|---|
| `Title` | Title | Single line of text | Ne | Ne | Da | Verovatno tekstualni identifikator rezervacije |
| `RezervisaniBroj` | Rezervisani broj | Number | Ne | Ne | Da | Numerički rezervisani broj |
| `DatumRezervacije` | DatumRezervacije | Date and Time | Ne | Ne | Da | Datum rezervacije broja |

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

Moguća namena:

```text
Tekstualni identifikator rezervacije, poslovni ključ ili opis rezervacije.
```

Status:

```text
POTVRĐENO: polje postoji.
NEPOZNATO: stvarna poslovna upotreba nije dostavljena.
```

Važna napomena:

```text
U dostavljenom XML-u Title nema uključeno EnforceUniqueValues.
```

To znači da, prema dostavljenom metadata odgovoru, `Title` trenutno nije SharePoint unique kolona.

---

## 5.2 RezervisaniBroj

SharePoint metadata:

```text
InternalName: RezervisaniBroj
DisplayName: Rezervisani broj
TypeAsString: Number
FieldTypeKind: 9
Decimals: 0
CommaSeparator: true
Required: false
ReadOnlyField: false
EnforceUniqueValues: false
Indexed: false
```

Namena:

```text
Čuvanje numeričkog rezervisanog broja.
```

Zaključak:

```text
Ovo je centralno polje za rezervaciju broja.
```

Rizik:

```text
U dostavljenom XML-u nije potvrđeno da je polje `RezervisaniBroj` unique.
```

Ako se sistem oslanja samo na `RezervisaniBroj` bez dodatne zaštite, postoji rizik od duplikata u paralelnom radu.

---

## 5.3 DatumRezervacije

SharePoint metadata:

```text
InternalName: DatumRezervacije
DisplayName: DatumRezervacije
TypeAsString: DateTime
FieldTypeKind: 4
DisplayFormat: DateOnly
Required: false
ReadOnlyField: false
EnforceUniqueValues: false
Indexed: false
```

Namena:

```text
Datum kada je broj rezervisan.
```

Moguća upotreba:

- grupisanje brojeva po datumu
- dnevni redni brojevi
- godišnji redni brojevi
- audit rezervacije
- pretraga rezervisanih brojeva

---

## 6. Poslovna uloga liste

Lista `RezervisaniBrojevi` služi kao mehanizam za kontrolu rezervacije delovodnih brojeva.

Korisnik je dodatno potvrdio:

```text
Za dodelu delovodnog broja postoji poseban flow zbog race condition problema koji se pojavi kada dvoje korisnika istovremeno zavode dokument.
```

Naziv potvrđenog flow-a:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Zaključak:

```text
RezervisaniBrojevi je deo mehanizma za sprečavanje duplog delovodnog broja, ali tačan nivo zaštite zavisi od implementacije flow-a.
```

---

## 7. Povezani Power Automate flow

Potvrđen flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Namena:

```text
Dodela / dodavanje delovodnog broja novom dokumentu iz Power Apps aplikacije.
```

Potvrđeno:

```text
Flow postoji zbog race condition problema.
```

Nepoznato:

```text
Nije dostavljena interna logika flow-a.
```

Zbog toga nije moguće potvrditi:

- da li flow koristi SharePoint unique constraint
- da li flow koristi atomic update
- da li flow koristi optimistic locking preko ETag-a
- da li flow ima retry logiku
- da li flow upisuje audit log
- da li flow vraća kontrolisanu grešku Canvas aplikaciji
- da li flow zaključava broj pre kreiranja dokumenta
- da li flow briše / oslobađa rezervaciju ako finalni upis ne uspe

---

## 8. Trenutni proces — logička interpretacija

Na osnovu naziva liste, naziva polja i potvrde korisnika, pretpostavljeni tok je:

```text
Canvas App
   |
   | korisnik pokreće zavođenje dokumenta
   v
Power Automate flow
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
   |
   | određuje / rezerviše sledeći broj
   v
RezervisaniBrojevi
   |
   | čuva rezervisani broj i datum
   v
Dokument / predmet dobija delovodni broj
```

Status:

```text
PRETPOSTAVKA zasnovana na dostavljenim podacima.
```

---

## 9. Kritični enterprise zahtev

Sistem mora podržati paralelno zavođenje.

Zahtev:

```text
Dva ili više korisnika mogu istovremeno kliknuti na zavođenje dokumenta.
```

Sistem mora garantovati:

```text
Samo jedan korisnik može dobiti konkretan delovodni broj.
```

Nedozvoljeno:

```text
Korisnik A dobije broj 100
Korisnik B istovremeno dobije broj 100
```

Ispravno:

```text
Korisnik A dobije broj 100
Korisnik B dobije broj 101
```

ili:

```text
Korisnik A dobije broj 100
Korisnik B prvi pokušaj ne uspe zbog konflikta
Sistem automatski ponovi pokušaj
Korisnik B dobije broj 101
```

---

## 10. Šta Canvas aplikacija ne sme da radi

Canvas aplikacija ne sme samostalno da garantuje jedinstvenost broja.

Nedozvoljen obrazac:

```powerfx
Set(
    varNextNumber,
    Max(RezervisaniBrojevi, RezervisaniBroj) + 1
)
```

Problem:

```text
Dva korisnika mogu u isto vreme pročitati isti Max broj.
```

Primer race condition-a:

```text
Trenutni max broj: 100

Korisnik A pročita 100
Korisnik B pročita 100

Korisnik A računa 101
Korisnik B računa 101

Oba pokušaju da upišu 101
```

Ako nema unique constraint i server-side kontrole, oba upisa mogu proći ili može nastati nekonzistentno stanje.

---

## 11. Šta Canvas aplikacija sme da radi

Canvas aplikacija sme da:

- prikaže formu
- validira obavezna polja
- pripremi payload
- pozove Power Automate flow
- prikaže spinner
- prikaže uspeh ili grešku
- osveži kolekcije nakon uspeha

Canvas aplikacija ne treba da:

- računa konačni delovodni broj
- rezerviše broj lokalno
- garantuje jedinstvenost
- rešava retry logiku
- rešava concurrency problem

---

## 12. Preporučeni backend obrazac

Za novu enterprise verziju preporučeni obrazac je:

```text
Power Apps
   |
   | Submit payload without final number
   v
Power Automate / backend service
   |
   | Get current counter / candidate number
   | Try reserve number
   | If conflict -> retry
   | If success -> create/update document
   v
SharePoint
   |
   | Unique delovodni broj
   | Reservation record
   | Audit log
```

---

## 13. Preporučeni concurrency-safe obrazac

Minimalni enterprise obrazac:

```text
1. Flow primi zahtev iz Canvas aplikacije.
2. Flow izračuna kandidat broj.
3. Flow pokuša da kreira rezervaciju.
4. SharePoint unique constraint odbija duplikat.
5. Ako upis uspe, broj je rezervisan.
6. Ako upis ne uspe zbog duplikata, flow ponavlja pokušaj.
7. Nakon uspešne rezervacije, flow upisuje delovodni broj na dokument.
8. Flow vraća rezultat aplikaciji.
9. Flow upisuje audit log.
```

---

## 14. Unique constraint

Preporuka:

```text
Na finalnom delovodnom broju mora postojati unique constraint.
```

Moguće lokacije:

| Lokacija | Polje | Preporuka |
|---|---|---|
| `RezervisaniBrojevi` | `Title` ili novi `ReservationKey` | Unique |
| `RezervisaniBrojevi` | `RezervisaniBroj` | Unique ako je globalni broj |
| Glavna lista predmeta / dokumenata | `DelovodniBroj` | Unique |
| Document library | `DelovodniBroj` | Index + kontrola, ako SharePoint dozvoljava za taj scenario |

Važno:

```text
Ako broj nije globalan nego zavisi od godine, tipa ili organizacione jedinice, unique ključ mora biti kompozitni.
```

SharePoint nema nativni multi-column unique constraint, pa se preporučuje posebno tekstualno polje.

Primer:

```text
ReservationKey = 2026|OS|100
```

ili:

```text
ReservationKey = 2026-OS-000100
```

Na `ReservationKey` se uključuje:

```text
Enforce unique values = Yes
```

---

## 15. Preporučeni dodatni model za rezervacije

Postojeća lista ima minimalna polja.

Za enterprise verziju preporučuje se proširenje ili nova lista:

```text
RegistryNumberReservations
```

Predložena polja:

| Internal name | Tip | Namena |
|---|---|---|
| `Title` | Text | Može biti isti kao `ReservationKey` |
| `ReservationKey` | Text | Jedinstveni ključ rezervacije |
| `RegistryYear` | Number | Godina |
| `RegistryPrefix` | Text | Prefix / tip broja |
| `ReservedNumber` | Number | Broj |
| `FormattedRegistryNumber` | Text | Finalni delovodni broj |
| `ReservationStatus` | Choice | Reserved / Used / Cancelled / Failed |
| `ReservedAt` | DateTime | Kada je rezervisano |
| `ReservedByEmail` | Text | Ko je inicirao |
| `SourceApp` | Text | Power Apps / Email / Import |
| `SourceDocumentId` | Text | ID dokumenta / fajla |
| `CorrelationId` | Text | ID flow run-a |
| `RetryCount` | Number | Broj pokušaja |
| `ErrorMessage` | Multiple lines | Greška ako postoji |

---

## 16. Preporučeni statusi rezervacije

| Status | Značenje |
|---|---|
| `Reserved` | Broj je rezervisan, ali dokument još nije finalizovan |
| `Used` | Broj je iskorišćen i povezan sa dokumentom |
| `Cancelled` | Rezervacija je poništena |
| `Expired` | Rezervacija je istekla bez finalizacije |
| `Failed` | Rezervacija ili finalni upis nisu uspeli |

---

## 17. Retry logika

Flow mora imati kontrolisan retry.

Preporučeni model:

```text
MaxRetryCount = 5
```

Tok:

```text
Attempt 1:
   Try number N

If conflict:
   Attempt 2:
      Try number N + 1

If conflict:
   Attempt 3:
      Try number N + 2
```

Ako ni nakon maksimalnog broja pokušaja ne uspe:

```text
Flow vraća kontrolisanu grešku u Power Apps.
```

Primer response payload-a:

```json
{
  "success": false,
  "errorCode": "REGISTRY_NUMBER_CONFLICT",
  "message": "Nije moguće rezervisati delovodni broj nakon maksimalnog broja pokušaja.",
  "retryCount": 5,
  "correlationId": "FLOW-RUN-ID"
}
```

---

## 18. Uspešan response prema Canvas aplikaciji

Primer:

```json
{
  "success": true,
  "reservedNumber": 101,
  "formattedRegistryNumber": "Os.Del.Br. 101/2026",
  "reservationId": 1234,
  "correlationId": "FLOW-RUN-ID"
}
```

Canvas aplikacija nakon toga treba da:

- prikaže korisniku dodeljen delovodni broj
- osveži dokument / kolekciju
- prikaže uspešnu poruku
- zaključa finalni broj za dalju ručnu izmenu

---

## 19. Greške koje treba kontrolisati

| Error code | Scenario |
|---|---|
| `REGISTRY_NUMBER_CONFLICT` | Broj već postoji |
| `MAX_RETRY_EXCEEDED` | Retry pokušaji potrošeni |
| `SHAREPOINT_WRITE_FAILED` | SharePoint upis nije uspeo |
| `DOCUMENT_UPDATE_FAILED` | Dokument nije ažuriran nakon rezervacije |
| `INVALID_PAYLOAD` | Power Apps nije poslao validne podatke |
| `PERMISSION_DENIED` | Service account nema pravo upisa |
| `CONFIG_NOT_FOUND` | Nema potrebne konfiguracije za numeraciju |
| `UNKNOWN_ERROR` | Neobrađena greška |

---

## 20. Audit log

Za svaki pokušaj rezervacije treba čuvati audit.

Minimalni audit:

| Polje | Opis |
|---|---|
| `CorrelationId` | ID flow run-a |
| `RequestedByEmail` | Korisnik koji je pokrenuo zavođenje |
| `RequestedAt` | Vreme zahteva |
| `CandidateNumber` | Broj koji je pokušao |
| `FinalNumber` | Broj koji je dodeljen |
| `Status` | Success / Failed / Retry |
| `RetryCount` | Broj pokušaja |
| `DocumentId` | ID dokumenta |
| `ErrorMessage` | Greška ako postoji |

Preporučena posebna lista:

```text
RegistryNumberAudit
```

---

## 21. Power Automate konfiguracija

Za flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

preporuke su:

1. Uključiti concurrency control tamo gde je potrebno.
2. Ako flow koristi trigger iz Power Apps, voditi računa da paralelni run-ovi ne naprave duplikat.
3. Kritični deo rezervacije mora imati server-side zaštitu.
4. Ne oslanjati se samo na globalno isključivanje paralelizma ako sistem treba da podrži više korisnika.
5. Unique constraint mora biti poslednja linija odbrane.
6. Svaki konflikt treba tretirati kao očekivan scenario, ne kao fatalnu grešku.
7. Response ka Power Apps mora biti standardizovan.

---

## 22. Važna arhitektonska napomena

Nije dovoljno samo podesiti Power Automate concurrency na `1`.

To bi sprečilo paralelnu obradu, ali bi moglo da napravi usko grlo.

Enterprise cilj je:

```text
Dozvoliti paralelne zahteve, ali atomarno dodeljivati jedinstvene brojeve.
```

To znači:

```text
Paralelnost korisnika: DA
Dupli delovodni broj: NIKADA
```

---

## 23. Indeksiranje

U dostavljenom XML-u nije potvrđeno da su ključna polja indeksirana:

```text
RezervisaniBroj Indexed: false
DatumRezervacije Indexed: false
Title Indexed: false
```

Preporuka:

Indeksirati polja koja se koriste za:

- pretragu poslednjeg broja
- filtriranje po godini
- filtriranje po datumu
- pronalaženje rezervacije
- proveru duplikata

Preporučena polja za indeks:

| Polje | Razlog |
|---|---|
| `ReservationKey` | Provera jedinstvenosti |
| `ReservedNumber` | Pretraga broja |
| `RegistryYear` | Godišnje numeracije |
| `RegistryPrefix` | Različite serije brojeva |
| `ReservationStatus` | Kontrola rezervacija |
| `Created` | Audit i sortiranje |

---

## 24. Poznati rizici

| Rizik | Opis | Uticaj |
|---|---|---|
| `RezervisaniBroj` nije unique | Moguć duplikat | Kritičan |
| `Title` nije unique | Nema zaštite na tekstualnom ključu | Visok |
| Nema potvrđenog audit log-a | Teško dokazivanje ko je dobio broj | Visok |
| Nema poznate retry logike | Konflikti mogu rušiti proces | Visok |
| Nema poznatog rollback-a | Rezervisani broj može ostati neiskorišćen | Srednji |
| Nema poznatog statusa rezervacije | Teško razlikovati Reserved i Used | Srednji |
| Nema potvrđene indeksacije | Performanse mogu pasti sa rastom liste | Srednji / visok |

---

## 25. Minimalni acceptance kriterijumi

Nova verzija mora ispuniti:

```text
AC-001:
Kada dva korisnika istovremeno zavode dokument, sistem ne sme dodeliti isti delovodni broj.

AC-002:
Ako prvi pokušaj rezervacije broja naiđe na konflikt, sistem mora automatski pokušati sledeći broj.

AC-003:
Ako rezervacija uspe, finalni delovodni broj mora biti upisan na dokument.

AC-004:
Ako finalni upis dokumenta ne uspe, rezervacija mora imati status koji omogućava audit i eventualni rollback.

AC-005:
Svaki pokušaj rezervacije mora imati audit zapis.

AC-006:
Finalni delovodni broj mora biti jedinstven na nivou definisanog poslovnog ključa.

AC-007:
Canvas aplikacija ne sme samostalno određivati konačni delovodni broj.

AC-008:
Flow mora vratiti standardizovan success/error response.
```

---

## 26. Test scenariji

### Test 1 — jedan korisnik

```text
1. Korisnik A zavodi dokument.
2. Flow dodeljuje sledeći broj.
3. Dokument dobija broj.
4. Rezervacija ima status Used.
```

Očekivanje:

```text
Uspešno.
```

---

### Test 2 — dva korisnika istovremeno

```text
1. Korisnik A i korisnik B istovremeno kliknu na zavođenje.
2. Oba zahteva stižu u flow.
3. Sistem dodeljuje različite brojeve.
```

Očekivanje:

```text
Korisnik A dobija broj N.
Korisnik B dobija broj N+1.
Duplikat nije moguć.
```

---

### Test 3 — konflikt pri rezervaciji

```text
1. Flow pokuša da rezerviše broj N.
2. SharePoint vrati grešku jer broj već postoji.
3. Flow pokušava broj N+1.
```

Očekivanje:

```text
Flow se oporavlja retry logikom.
```

---

### Test 4 — finalni upis dokumenta ne uspe

```text
1. Broj je rezervisan.
2. Ažuriranje dokumenta ne uspe.
3. Flow mora ostaviti audit i status greške.
```

Očekivanje:

```text
Rezervacija nije izgubljena.
Greška je vidljiva administratoru.
```

---

## 27. Preporuke za novu enterprise verziju

1. Uvesti `ReservationKey` kao jedinstvenu tekstualnu kolonu.
2. Uključiti unique constraint na `ReservationKey`.
3. Uvesti `ReservationStatus`.
4. Uvesti `CorrelationId`.
5. Uvesti `RetryCount`.
6. Uvesti posebnu audit listu.
7. Finalnu dodelu broja držati isključivo u Power Automate / backend sloju.
8. Canvas aplikaciju koristiti samo kao UI i inicijator procesa.
9. Uvesti standardizovane error kodove.
10. Dokumentovati sve edge case scenarije.
11. Testirati paralelno zavođenje sa više korisnika.
12. Uvesti indeksiranje ključnih polja.
13. Razmotriti Dataverse ili Azure backend ako sistem značajno poraste.

---

## 28. Otvorena pitanja

| Pitanje | Status |
|---|---|
| Da li je `RezervisaniBroj` trenutno unique? | NEPOZNATO |
| Da li postoji dodatno polje za finalni format delovodnog broja? | NEPOZNATO |
| Da li se broj resetuje dnevno, mesečno, godišnje ili nikada? | NEPOZNATO |
| Da li postoji prefix po tipu dokumenta? | NEPOZNATO |
| Da li postoji prefix po organizacionoj jedinici? | NEPOZNATO |
| Da li flow koristi ETag / optimistic locking? | NEPOZNATO |
| Da li flow koristi SharePoint unique constraint? | NEPOZNATO |
| Da li postoji audit lista za rezervacije? | NEPOZNATO |
| Da li se rezervacija oslobađa ako upload dokumenta ne uspe? | NEPOZNATO |
| Koji korisnički identitet upisuje rezervaciju — korisnik ili service account? | NEPOZNATO |
| Da li `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` vraća JSON response? | NEPOZNATO |
| Koji je format finalnog delovodnog broja? | NEPOZNATO |

---

## 29. Zaključak

`RezervisaniBrojevi` je jedna od najvažnijih lista u rešenju DocCentral v6.0.

Njena uloga je direktno povezana sa najkritičnijim enterprise zahtevom:

```text
Nikada ne sme doći do duplog delovodnog broja.
```

Potvrđeno je da postoji poseban flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

i da je uveden zbog race condition problema.

Međutim, iz dostavljenog XML-a nije potvrđeno da trenutna lista ima uključene unique constraint i indeksiranje na ključnim poljima.

Zbog toga je glavna preporuka:

```text
U novoj verziji implementirati centralizovan, auditovan i concurrency-safe servis za rezervaciju delovodnih brojeva, sa unique constraint zaštitom kao poslednjom linijom odbrane.
```

---

## 30. Povezani dokumenti

```text
README.md
docs/03-sharepoint-data-model.md
docs/06-numbering-and-concurrency.md
docs/08-power-automate-analysis.md
docs/10-known-issues-technical-debt.md
docs/11-enterprise-improvements.md
architecture/numbering-sequence.md
data-model/appconfig.md
data-model/shared-documents.md
backlog/known-issues.md
backlog/enterprise-backlog.md
backlog/open-questions.md
```
