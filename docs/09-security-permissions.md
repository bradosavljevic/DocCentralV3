# 09 — Security i permission model

## 1. Svrha dokumenta

Ovaj dokument opisuje trenutno poznati security i permission model rešenja **DocCentral v6.0**.

Cilj dokumenta je da zabeleži:

- poznate činjenice o pristupu podacima
- ulogu Canvas aplikacije
- ulogu Power Automate flow-ova
- odnos korisnika prema SharePoint listama i bibliotekama
- rizike u trenutnom modelu
- preporuke za enterprise verziju
- otvorena pitanja za dopunu dokumentacije

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Power Apps Canvas aplikacija | POTVRĐENO |
| SharePoint Online kao data/storage layer | POTVRĐENO |
| Power Automate kao backend sloj | POTVRĐENO |
| Upload dokumenta preko Power Automate-a | POTVRĐENO |
| Poseban flow za dodelu delovodnog broja | POTVRĐENO |
| Email intake preko shared mailbox-a | POTVRĐENO na nivou funkcionalnosti |
| AppConfig kao konfiguraciona lista | POTVRĐENO |
| Tačne SharePoint permission grupe | NEPOZNATO |
| Da li korisnici imaju direktna Write prava na SharePoint | NEPOZNATO za ovo rešenje |
| Da li flow-ovi rade pod service account-om | NEPOZNATO |
| Da li postoji item-level permission model | NEPOZNATO |
| Da li se prava lome po dokumentu | NEPOZNATO |
| Da li se koristi audit log lista | NEPOZNATO |
| Da li se koristi Entra ID security group model | NEPOZNATO |
| Da li postoji admin rola u aplikaciji | NEPOZNATO |

---

## 3. Visok nivo security modela

Trenutni model, na osnovu poznatih informacija, može se opisati ovako:

```text
Korisnik
   |
   v
Power Apps Canvas App
   |
   | poziva flow-ove
   v
Power Automate
   |
   | kreira / ažurira / uploaduje / dodeljuje brojeve
   v
SharePoint Online
   |
   | liste + biblioteke
   v
Dokumenti, konfiguracija, delovodni brojevi, exporti
```

U ovom modelu Power Automate treba da bude kontrolisana backend tačka za operacije koje ne smeju zavisiti samo od klijentske logike u Canvas aplikaciji.

---

## 4. Ključni principi security modela

Za enterprise verziju DocCentral rešenja treba primeniti sledeće principe:

1. **Least privilege** — korisnici treba da imaju samo minimalna prava koja su im potrebna za rad.
2. **Backend-controlled writes** — kritični upisi treba da idu kroz Power Automate, ne direktno iz Canvas aplikacije.
3. **Centralizovana validacija** — Power Automate mora da validira podatke pre upisa u SharePoint.
4. **Auditability** — kritične akcije moraju imati audit trag.
5. **Separation of duties** — administracija konfiguracije, unos dokumenata i sistemsko održavanje treba da budu razdvojeni.
6. **Concurrency safety** — security model mora podržati istovremeni rad više korisnika bez duplih delovodnih brojeva.

---

## 5. Korisnički pristup

### 5.1 Poznato

Korisnici rade kroz Canvas aplikaciju.

Početni ekran aplikacije:

```text
scrHome
```

Postoji poseban ekran za zavođenje dokumenta.

Pregled dokumenata se ne radi kroz poseban ekran u aplikaciji, već direktno kroz SharePoint listu.

### 5.2 Implikacija

Pošto se pregled dokumenata radi direktno kroz SharePoint, korisnici moraju imati barem Read prava nad relevantnim SharePoint listama i/ili bibliotekama.

To znači da SharePoint permission model nije samo tehnička pozadina, već direktno utiče na korisničko iskustvo.

### 5.3 Nepoznato

Potrebno je potvrditi:

- koje korisničke grupe postoje
- ko ima pristup Canvas aplikaciji
- ko ima pristup SharePoint site-u
- ko ima pristup glavnoj biblioteci dokumenata
- ko ima pristup `EmailDocuments`
- ko ima pristup `Exports`
- ko ima pristup `AppConfig`
- ko ima pristup `RezervisaniBrojevi`
- da li korisnici imaju Read, Contribute, Edit ili Full Control prava
- da li postoji poseban admin role model

---

## 6. SharePoint elementi i security zahtevi

### 6.1 AppConfig

Lista:

```text
AppConfig
```

Namena:

- centralna konfiguracija aplikacije
- šifarnici
- JSON konfiguracije
- konfiguracija kolona
- export konfiguracije

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| Title | Title | Single line of text |
| Config | Config | Multiple lines of text |
| ColumnHeader | ColumnHeader | Multiple lines of text |
| ID | ID | Counter |
| Created | Created | Date and Time |
| Modified | Modified | Date and Time |
| Author | Created By | Person or Group |
| Editor | Modified By | Person or Group |

#### Security preporuka

`AppConfig` ne treba da bude otvoren za izmenu svim korisnicima.

Preporučeni model:

| Rola | Prava |
|---|---|
| Običan korisnik | Read |
| Referent / operater | Read |
| Power user | Read |
| Administrator aplikacije | Edit |
| Service account / flow owner | Edit |
| Developer / support | Po potrebi, kontrolisano |

#### Rizik

Ako običan korisnik ima Edit prava nad `AppConfig`, može promeniti:

- šifarnike
- JSON konfiguraciju
- pravila aplikacije
- prikaz kolona
- export podešavanja
- potencijalno ponašanje aplikacije

To je visok rizik.

---

### 6.2 RezervisaniBrojevi

Lista:

```text
RezervisaniBrojevi
```

Namena:

- rezervacija / kontrola delovodnih brojeva
- potencijalno deo concurrency-safe mehanizma

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| Title | Title | Single line of text |
| RezervisaniBroj | Rezervisani broj | Number |
| DatumRezervacije | DatumRezervacije | Date and Time |
| ID | ID | Counter |
| Created | Created | Date and Time |
| Modified | Modified | Date and Time |
| Author | Created By | Person or Group |
| Editor | Modified By | Person or Group |

#### Security preporuka

`RezervisaniBrojevi` treba da bude backend-controlled lista.

Preporučeni model:

| Rola | Prava |
|---|---|
| Običan korisnik | No direct write |
| Referent / operater | No direct write |
| Canvas aplikacija | Ne treba direktno da garantuje broj |
| Power Automate flow | Create/Edit |
| Administrator | Read/Edit, kontrolisano |
| Auditor | Read |

#### Kritična napomena

Korisnici ne treba ručno da menjaju stavke u ovoj listi.

Ako korisnik može ručno da promeni rezervisani broj, narušava se integritet delovodnog broja.

---

### 6.3 Shared Documents

Biblioteka:

```text
Shared Documents
```

Namena:

- glavna biblioteka dokumenata
- čuvanje dokumenata povezanih sa delovodnim brojem

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| FileLeafRef | Name | File |
| Title | Title | Single line of text |
| _ExtendedDescription | Description | Multiple lines of text |
| DelovodniBroj | DelovodniBroj | Single line of text |
| EdokumentID | EdokumentID | Single line of text |
| Edokument | Edokument | Yes/No |
| Attachment | Attachment | Yes/No |
| MediaServiceImageTags | Image Tags | Managed Metadata |
| ContentType | Content Type | Computed |

#### Security preporuka

Pristup dokumentima treba definisati po poslovnim pravilima.

Minimalni preporučeni model:

| Rola | Prava |
|---|---|
| Običan korisnik | Read samo za dokumente koje sme da vidi |
| Referent / operater | Read, upload preko flow-a |
| Power Automate flow | Create/Edit |
| Administrator | Full Control ili Edit |
| Auditor | Read |
| Service account | Owner/Edit, zavisno od dizajna |

#### Rizici

- Ako svi korisnici imaju Edit prava, mogu menjati metadata i dokumente mimo aplikacije.
- Ako svi korisnici imaju Delete prava, postoji rizik gubitka dokumenata.
- Ako nema audit loga, teško je dokazati ko je promenio dokument.
- Ako `DelovodniBroj` nije zaštićen, korisnik može ručno izmeniti vezu dokumenta i broja.

---

### 6.4 EmailDocuments

Biblioteka:

```text
EmailDocuments
```

Namena:

- dokumenti dobijeni ili obrađeni iz shared mailbox procesa

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| FileLeafRef | Name | File |
| Title | Title | Single line of text |
| _ExtendedDescription | Description | Multiple lines of text |
| PosiljalacEmail | PosiljalacEmail | Single line of text |
| Posiljalac | Posiljalac | Single line of text |
| MediaServiceImageTags | Image Tags | Managed Metadata |
| ContentType | Content Type | Computed |

#### Security preporuka

Ova biblioteka treba da bude kontrolisana Power Automate procesom.

Preporučeni model:

| Rola | Prava |
|---|---|
| Običan korisnik | Read samo ako poslovno treba |
| Referent / operater | Read / obrada kroz aplikaciju |
| Email processing flow | Create/Edit |
| Administrator | Edit |
| Auditor | Read |

#### Rizik

Email dokumenti mogu sadržati osetljive informacije.

Ako je biblioteka preširoko otvorena, korisnici mogu videti dokumente koje ne bi trebalo da vide.

---

### 6.5 Exports

Biblioteka:

```text
Exports
```

Namena:

- export šifarnika
- export konfiguracije iz `AppConfig`
- export fajlovi / izveštaji

Poznata polja:

| Internal name | Display name | Tip |
|---|---|---|
| FileLeafRef | Name | File |
| Title | Title | Single line of text |
| _ExtendedDescription | Description | Multiple lines of text |
| ContentType | Content Type | Computed |

#### Security preporuka

`Exports` može sadržati konfiguracione podatke, pa ne treba biti javno otvoren svim korisnicima.

Preporučeni model:

| Rola | Prava |
|---|---|
| Običan korisnik | No access ili Read samo po potrebi |
| Administrator aplikacije | Read/Edit |
| Export flow | Create/Edit |
| Developer / support | Read po potrebi |
| Auditor | Read |

#### Rizik

Ako export sadrži `AppConfig`, može sadržati poslovna pravila, interne šifarnike ili konfiguraciju aplikacije.

---

## 7. Power Automate security model

### 7.1 Poznato

Power Automate se koristi za:

- upload dokumenta
- dodelu delovodnog broja
- email intake iz shared mailbox-a
- export konfiguracije i šifarnika

Poznati flow:

```text
Upload Doc
```

Poznati flow za dodelu broja:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

### 7.2 Security preporuka

Flow-ovi treba da rade pod kontrolisanim nalogom, idealno service account-om.

Prednosti service account modela:

- kontrolisano vlasništvo nad flow-ovima
- stabilnije konekcije
- lakše održavanje
- manje zavisnosti od pojedinačnih korisnika
- centralizovana prava
- lakši audit

### 7.3 Rizik ako flow radi pod ličnim nalogom

Ako flow radi pod ličnim nalogom korisnika/developera:

- flow može prestati da radi ako korisnik ode iz firme
- password / MFA / Conditional Access može prekinuti konekcije
- nije jasno ko je sistemski vlasnik
- audit može biti zbunjujući
- promena licence može uticati na rad flow-a

### 7.4 Nepoznato

Potrebno je potvrditi:

- ko je owner flow-ova
- pod kojim nalogom rade konekcije
- da li postoji service account
- koje konekcije koriste flow-ovi
- da li flow koristi SharePoint connector
- da li flow koristi Outlook / Exchange connector
- da li flow koristi premium konektore
- da li flow koristi child flow pattern
- da li postoje connection references u solution-u
- da li su connection references pravilno dokumentovane

---

## 8. Canvas app security model

### 8.1 Poznato

Aplikacija je Canvas App.

Početni ekran:

```text
scrHome
```

Poznato je da postoji poseban ekran za zavođenje dokumenta.

### 8.2 Preporuka

Canvas aplikacija treba da radi sledeće:

- prikaže ekran prema roli korisnika
- validira formu pre slanja
- pozove odgovarajući Power Automate flow
- prikaže odgovor flow-a
- ne garantuje sama finalni delovodni broj
- ne radi kritične upise ako korisnik ne treba da ima direktna SharePoint Write prava

### 8.3 Rizik

Canvas aplikacija nije bezbedno mesto za čuvanje kritične poslovne logike ako korisnik može da zaobiđe aplikaciju i direktno menja SharePoint podatke.

Zato security mora biti postavljen na SharePoint / Power Automate sloju.

---

## 9. Predlog rola

Za enterprise verziju preporučuje se definisanje sledećih rola.

| Rola | Opis |
|---|---|
| DocCentral User | Osnovni korisnik koji vidi dokumente prema pravilima |
| DocCentral Clerk | Korisnik koji zavodi dokumente |
| DocCentral Email Processor | Tehnička rola / flow za email dokumente |
| DocCentral Config Admin | Korisnik koji menja konfiguraciju i šifarnike |
| DocCentral Auditor | Korisnik koji ima read-only pristup audit tragovima |
| DocCentral System Account | Service account za flow-ove i backend operacije |
| DocCentral Developer / Support | Tehnička podrška i razvoj |

---

## 10. Predlog permission matrice

| Element | User | Clerk | Config Admin | Auditor | Flow / Service Account |
|---|---:|---:|---:|---:|---:|
| Canvas App | Use | Use | Use | Optional | N/A |
| AppConfig | Read | Read | Edit | Read | Edit |
| RezervisaniBrojevi | No direct write | No direct write | Read | Read | Create/Edit |
| Shared Documents | Read po pravilima | Read / upload preko flow-a | Read/Edit | Read | Create/Edit |
| EmailDocuments | Read po potrebi | Read / obrada | Read/Edit | Read | Create/Edit |
| Exports | No access / Read po potrebi | Read po potrebi | Read/Edit | Read | Create/Edit |
| Audit log | No access | Read po potrebi | Read | Read | Create |
| Error log | No access | Read po potrebi | Read | Read | Create |

---

## 11. Delovodni broj i security

### 11.1 Kritični zahtev

Najvažniji security/integrity zahtev:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

### 11.2 Security implikacija

Ovaj zahtev nije samo pitanje logike, već i pitanje prava.

Ako korisnici imaju mogućnost da direktno menjaju:

- `DelovodniBroj`
- `RezervisaniBroj`
- dokument metadata
- counter / rezervacione zapise

onda mogu slučajno ili namerno narušiti jedinstvenost brojeva.

### 11.3 Preporuka

Korisnici ne treba direktno da upravljaju finalnim delovodnim brojem.

Finalnu dodelu broja treba da radi:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

ili novi enterprise backend flow/service koji implementira:

- atomic reservation
- optimistic locking
- retry
- unique constraint
- audit log
- error response

---

## 12. Audit log preporuka

Za enterprise verziju preporučuje se posebna audit log lista.

Primer naziva:

```text
DocCentralAuditLog
```

Preporučena polja:

| Polje | Tip | Opis |
|---|---|---|
| Title | Text | Kratak naziv akcije |
| ActionType | Choice/Text | Tip akcije |
| ActorEmail | Text | Korisnik koji je inicirao akciju |
| ActorDisplayName | Text | Ime korisnika |
| FlowName | Text | Naziv flow-a |
| DocumentId | Number/Text | ID dokumenta |
| DelovodniBroj | Text | Dodeljeni broj |
| RequestId | Text | Jedinstveni ID zahteva |
| Status | Choice/Text | Success / Failed |
| ErrorCode | Text | Kod greške |
| ErrorMessage | Multiple lines | Detalj greške |
| PayloadJson | Multiple lines | Ulazni/izlazni JSON |
| CreatedAt | DateTime | Vreme akcije |

### 12.1 Šta treba logovati

Obavezno logovati:

- pokušaj dodele delovodnog broja
- uspešnu dodelu delovodnog broja
- neuspešnu dodelu delovodnog broja
- upload dokumenta
- obradu email attachmenta
- export konfiguracije
- izmenu AppConfig stavke
- greške flow-ova

---

## 13. Error log preporuka

Pored audit loga, preporučuje se i error log.

Primer naziva:

```text
DocCentralErrorLog
```

Preporučena polja:

| Polje | Tip | Opis |
|---|---|---|
| Title | Text | Kratak opis greške |
| FlowName | Text | Naziv flow-a |
| ActionName | Text | Akcija u flow-u |
| ErrorCode | Text | Kod greške |
| ErrorMessage | Multiple lines | Poruka greške |
| RawErrorJson | Multiple lines | Kompletan error payload |
| RequestPayloadJson | Multiple lines | Ulazni podaci |
| UserEmail | Text | Korisnik koji je inicirao proces |
| DocumentId | Text/Number | Dokument |
| DelovodniBroj | Text | Delovodni broj |
| CreatedAt | DateTime | Vreme greške |

---

## 14. Preporučeni service account model

Za enterprise rešenje preporučuje se poseban nalog.

Primer:

```text
dcadmin@tenant.onmicrosoft.com
```

ili klijentski definisan naziv:

```text
doccentral.service@domain.com
```

### 14.1 Service account treba da ima

- licencu potrebnu za pokretanje flow-ova
- pristup SharePoint site-u
- potrebna prava nad listama i bibliotekama
- pristup shared mailbox-u ako obrađuje email
- ownership ili co-ownership nad flow-ovima
- stabilne connection reference
- dokumentovanu namenu

### 14.2 Service account ne treba da bude

- lični nalog developera
- nalog zaposlenog koji nije sistemski vlasnik
- global admin bez potrebe

---

## 15. Preporuke za SharePoint permission dizajn

### 15.1 Site-level

Na nivou site-a preporuka je:

- minimalan broj site owner-a
- jasno definisane SharePoint grupe
- bez direktnog dodavanja pojedinačnih korisnika kad god je moguće
- korišćenje Entra ID security grupa gde je moguće

### 15.2 List/library-level

Za kritične liste i biblioteke:

- `AppConfig` — ograničiti Edit prava
- `RezervisaniBrojevi` — zabraniti korisnički direktan write
- `Shared Documents` — kontrolisati Edit/Delete
- `EmailDocuments` — ograničiti pristup zbog potencijalno osetljivih email podataka
- `Exports` — ograničiti zbog konfiguracionih exporta

### 15.3 Item/document-level

Ako poslovni proces zahteva da korisnici vide samo određene dokumente, moguće opcije su:

- SharePoint item-level permissions
- biblioteke/folderi po organizacionim celinama
- metadata security model
- security trimming kroz posebne view-eve
- Power Apps filtriranje uz backend validaciju
- Dataverse kao bolji enterprise data layer za kompleksan security

### 15.4 Rizik item-level permissions modela

Ako se prava lome za veliki broj dokumenata, mogu nastati problemi:

- sporiji rad
- kompleksnije održavanje
- teže troubleshooting
- veći rizik grešaka u nasleđivanju prava
- zahtevniji flow-ovi za grant/revoke access

---

## 16. Security rizici

### 16.1 Direktan SharePoint write

Ako korisnici imaju direktan Write/Edit pristup listama i bibliotekama, mogu zaobići aplikaciju i flow-ove.

Rizik:

- izmena delovodnog broja
- izmena metadata
- brisanje dokumenata
- izmena konfiguracije
- ručno kreiranje stavki mimo validacije

### 16.2 Slaba kontrola AppConfig liste

Ako `AppConfig` nije zaštićen, korisnik može promeniti ponašanje aplikacije.

### 16.3 Slaba kontrola RezervisaniBrojevi liste

Ako korisnik može direktno menjati `RezervisaniBrojevi`, može ugroziti jedinstvenost delovodnih brojeva.

### 16.4 Flow pod ličnim nalogom

Ako flow-ovi rade pod ličnim nalogom, rešenje zavisi od tog korisnika.

### 16.5 Nedostatak audit loga

Bez audit loga nije moguće pouzdano pratiti:

- ko je pokrenuo zavođenje
- ko je dobio koji broj
- kada je broj dodeljen
- da li je bilo grešaka
- ko je promenio konfiguraciju

### 16.6 Email intake rizici

Shared mailbox proces može imati dodatne rizike:

- duplo procesiranje emaila
- obrada malicioznih attachmenta
- obrada pogrešnog foldera
- izostanak evidencije ko je poslao dokument
- nejasna pravila za više attachmenta

---

## 17. Preporuke za novu enterprise verziju

### 17.1 Obavezno

- Uvesti jasan role model.
- Uvesti service account za flow-ove.
- Ograničiti korisnički direktan Write pristup kritičnim listama.
- Finalni delovodni broj dodeljivati isključivo backend logikom.
- Uvesti audit log.
- Uvesti error log.
- Uvesti unique constraint gde je tehnički moguće.
- Dokumentovati connection references.
- Dokumentovati vlasništvo flow-ova.
- Dokumentovati permission matrix.

### 17.2 Preporučeno

- Koristiti Entra ID security grupe.
- Razdvojiti admin, clerk, user i auditor role.
- Razmotriti Dataverse za kompleksniji enterprise security model.
- Razmotriti Application Insights ili centralizovani logging ako je dostupno.
- Uvesti idempotency key za kritične zahteve iz Canvas aplikacije.
- Uvesti kontrolu duplog klika u aplikaciji i backend zaštitu u flow-u.
- Uvesti environment variables za konfiguraciju site URL-ova i biblioteka.
- Uvesti ALM proces za solution deployment.

---

## 18. Otvorena pitanja

Potrebno je potvrditi sledeće:

1. Koje SharePoint grupe postoje na site-u?
2. Ko je Site Owner?
3. Ko su Members?
4. Ko su Visitors?
5. Da li korisnici koji zavode dokumenta imaju direktna Edit prava na SharePoint?
6. Da li korisnici imaju Delete prava?
7. Da li je `AppConfig` zaključan samo za administratore?
8. Da li obični korisnici mogu menjati `RezervisaniBrojevi`?
9. Ko je owner flow-a `Upload Doc`?
10. Ko je owner flow-a `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`?
11. Da li flow-ovi rade pod service account-om?
12. Koji shared mailbox koristi email intake proces?
13. Da li email intake flow ima pristup mailbox-u preko ličnog naloga ili service account-a?
14. Da li se koristi audit log lista?
15. Da li se koristi error log lista?
16. Da li postoji rola administratora u Canvas aplikaciji?
17. Da li se UI menja prema korisničkoj roli?
18. Da li se koriste Entra ID security grupe?
19. Da li se lome prava po dokumentu?
20. Da li se koristi `Grant access to an item or folder` akcija?
21. Da li postoji proces za offboarding korisnika?
22. Da li postoji proces za transfer vlasništva flow-ova?
23. Da li postoji dokumentovana permission matrica kod klijenta?

---

## 19. Zaključak

Security i permission model je jedan od najvažnijih delova DocCentral rešenja.

Najkritičniji zahtev je da više korisnika može istovremeno da zavodi dokumenta, ali da sistem nikada ne dozvoli isti delovodni broj za dva dokumenta.

Zato security model mora biti projektovan zajedno sa backend logikom za dodelu brojeva.

Najvažnije preporuke:

- korisnici ne treba direktno da kontrolišu finalni delovodni broj
- `RezervisaniBrojevi` mora biti zaštićena lista
- `AppConfig` mora biti zaštićena lista
- Power Automate flow-ovi treba da budu centralni sloj za kritične upise
- flow-ovi treba da rade pod kontrolisanim service account-om
- mora postojati audit log
- mora postojati error log
- mora postojati jasna permission matrica
- nova verzija mora imati enterprise-grade concurrency i security dizajn
