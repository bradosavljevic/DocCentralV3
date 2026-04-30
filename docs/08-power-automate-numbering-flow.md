# Poseban flow za delovodni broj

**Status:** POTVRĐENO  
**Naziv flow-a:** `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`

## 1. Svrha flow-a

Pored flow-a `Upload Doc`, postoji poseban Power Automate flow koji je namenjen rešavanju race condition problema pri istovremenom zavođenju dokumenata.

Flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

ima kritičnu ulogu u procesu zavođenja dokumenata jer mora obezbediti da dva korisnika nikada ne dobiju isti delovodni broj.

## 2. Poslovni razlog

U sistemu više korisnika mora moći istovremeno da zavodi dokumente.

Bez centralizovane kontrole može doći do situacije da dva korisnika u isto vreme:

1. pročitaju isti poslednji delovodni broj,
2. izračunaju isti sledeći broj,
3. pokušaju da zavode različite dokumente sa istim delovodnim brojem.

Zbog toga delovodni broj ne sme biti generisan samo u Canvas aplikaciji.

## 3. Odgovornost flow-a

Flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` odgovoran je za:

- generisanje ili rezervaciju sledećeg delovodnog broja,
- sprečavanje duplog delovodnog broja,
- kontrolu konkurentnog pristupa,
- backend proveru pre dodele broja,
- vraćanje jedinstvenog broja Canvas aplikaciji ili pozivajućem procesu,
- zaštitu procesa od race condition problema.

## 4. Odnos prema flow-u `Upload Doc`

Flow `Upload Doc` nije odgovoran za rešavanje problema konkurentnog generisanja delovodnog broja.

Za taj deo postoji poseban flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

To znači da se arhitektura može posmatrati ovako:

```text
Canvas App
    |
    | korisnik pokreće zavođenje dokumenta
    v
Power Automate flow za delovodni broj
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
    |
    | generiše / rezerviše jedinstven broj
    v
Upload Doc flow
    |
    | upload dokumenta i upis metapodataka
    v
SharePoint Online
```

## 5. Veza sa listom `RezervisaniBrojevi`

Na osnovu dostavljenih SharePoint metadata podataka, postoji lista:

```text
RezervisaniBrojevi
```

Identifikovana ključna polja:

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

## 6. Zaključak o listi `RezervisaniBrojevi`

**Pretpostavka:**  
Flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` koristi listu `RezervisaniBrojevi` kao deo mehanizma za rezervaciju delovodnog broja.

**Status:** PRETPOSTAVKA  
**Razlog:** naziv liste i struktura polja ukazuju na rezervaciju brojeva, ali konkretna interna logika flow-a još nije dostavljena.

## 7. Kritičan enterprise zahtev

Najvažniji zahtev za ovaj deo sistema je:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovo je enterprise zahtev najvišeg prioriteta.

## 8. Pravila za novu verziju

U novoj verziji rešenja logika delovodnog broja mora biti posebno projektovana, testirana i dokumentovana.

Obavezna pravila:

- delovodni broj ne sme biti generisan samo u Canvas aplikaciji,
- delovodni broj ne sme zavisiti od lokalne Power Apps kolekcije,
- finalna dodela broja mora biti urađena server-side,
- mora postojati centralizovan servis ili flow za brojanje,
- mora postojati atomic ili optimistic locking mehanizam,
- mora postojati retry logika,
- mora postojati unique constraint na finalnom polju `DelovodniBroj`,
- mora postojati audit log svake rezervacije broja,
- mora postojati log neuspelih pokušaja,
- korisnik mora dobiti kontrolisanu grešku ako rezervacija broja nije uspela.

## 9. Preporučeni target dizajn

Preporučeni target dizajn za enterprise verziju:

```text
Canvas App
    |
    | Submit request
    v
Numbering Service / Power Automate flow
    |
    | Lock / reserve number
    | Validate uniqueness
    | Retry if conflict
    v
SharePoint list / Dataverse table for counter
    |
    | Return confirmed number
    v
Document creation / Upload Doc
    |
    | Save document metadata
    v
Final SharePoint document item
```

## 10. Obavezni test scenariji

Za ovaj flow moraju se definisati posebni testovi:

### Test 1 — Jedan korisnik

Jedan korisnik zavodi jedan dokument.

Očekivanje:

- sistem uspešno generiše delovodni broj,
- dokument dobija jedinstven broj,
- nema greške.

### Test 2 — Dva korisnika istovremeno

Dva korisnika u isto vreme zavode dokument.

Očekivanje:

- oba dokumenta se uspešno zavode,
- svaki dokument dobija različit delovodni broj,
- nema duplikata.

### Test 3 — Više korisnika istovremeno

Pet ili više korisnika u isto vreme zavode dokumente.

Očekivanje:

- sistem dodeljuje jedinstvene brojeve,
- nema preskakanja osim ako je to poslovno prihvatljivo,
- nema duplikata,
- neuspešni pokušaji se loguju.

### Test 4 — Namerni konflikt

Simulirati situaciju gde dva procesa pokušavaju isti broj.

Očekivanje:

- jedan proces uspeva,
- drugi proces ulazi u retry,
- sistem dodeljuje novi broj,
- nema duplikata.

### Test 5 — Greška u rezervaciji

Simulirati grešku u toku rezervacije broja.

Očekivanje:

- korisnik dobija jasnu poruku,
- sistem ne kreira dokument sa nevalidnim brojem,
- greška se loguje.

## 11. Otvorena pitanja

Potrebno je dodatno potvrditi:

1. Da li flow `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` zaista koristi listu `RezervisaniBrojevi`?
2. Da li flow koristi SharePoint ETag / optimistic concurrency?
3. Da li postoji retry logika?
4. Da li postoji posebna audit lista?
5. Da li je `DelovodniBroj` podešen kao unique field u finalnoj listi ili biblioteci?
6. Da li se broj prvo rezerviše, pa tek onda dodeljuje dokumentu?
7. Šta se dešava sa rezervisanim brojem ako upload dokumenta ne uspe?
8. Da li postoji cleanup proces za neiskorišćene rezervisane brojeve?

## 12. Status dokumenta

Ovaj dokument je deo tehničke dokumentacije za DocCentral v6.0.

Preporučena lokacija u GitHub repozitorijumu:

```text
docs/06-numbering-and-concurrency.md
```

ili kao dodatni detaljni dokument:

```text
docs/08-power-automate-numbering-flow.md
```
