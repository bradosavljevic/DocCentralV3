# DocCentral v6.0

## Tehnička dokumentacija, pravila analize i Claude Code razvojna osnova

Ovaj repozitorijum sadrži dokumentaciju i razvojna uputstva za **DocCentral v6.0**, Power Platform rešenje za elektronsku pisarnicu, evidenciju dokumenata i rad sa SharePoint listama/bibliotekama.

Repozitorijum trenutno nije namenjen za čuvanje kompletnog Power Platform solution fajla kao glavnog artefakta, već za:

- tehničku dokumentaciju,
- uputstva za analizu,
- SharePoint data model dokumentaciju,
- Claude Code / Cloud Code skillove,
- šablone za buduću dokumentaciju,
- checkliste za kontrolu kvaliteta,
- razvojni brief za novu enterprise verziju aplikacije.

---

## 1. Svrha repozitorijuma

Cilj ovog repozitorijuma je da bude centralno mesto za razumevanje postojećeg DocCentral rešenja i pripremu nove unapređene verzije.

Dokumentacija treba da omogući:

1. Razumevanje poslovne svrhe aplikacije.
2. Razumevanje trenutne Power Platform arhitekture.
3. Dokumentovanje SharePoint data modela.
4. Dokumentovanje Power Apps Canvas aplikacije kada solution bude analiziran.
5. Dokumentovanje Power Automate flow-ova kada definicije budu dostupne.
6. Identifikaciju rizika, tehničkog duga i nepoznatih delova.
7. Definisanje enterprise unapređenja.
8. Pripremu jasnog inputa za Claude Code / Cloud Code razvoj nove verzije.

---

## 2. Osnovni opis rešenja

**DocCentral v6.0** je Power Platform rešenje za elektronsku pisarnicu i centralnu evidenciju dokumenata.

Na osnovu dosadašnjih SharePoint REST XML exporta i dostavljenih objašnjenja, rešenje koristi:

- Power Apps Canvas aplikaciju kao korisnički interfejs,
- Power Automate flow-ove za backend logiku,
- SharePoint Online liste kao data model,
- SharePoint Online biblioteke kao storage layer za dokumente,
- konfiguracione liste za podešavanja aplikacije,
- liste i/ili flow logiku za generisanje i kontrolu delovodnih brojeva.

---

## 3. SharePoint okruženje

### Site URL

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

### REST API base URL

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 4. Ključni enterprise zahtev

Najvažniji zahtev za novu verziju DocCentral rešenja:

> Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovo je kritičan poslovni, pravni i arhitekturni zahtev.

### Posledice po arhitekturu

Delovodni broj ne sme biti garantovan samo kroz Canvas aplikaciju, lokalne kolekcije ili čitanje poslednjeg broja iz SharePoint liste bez kontrole konkurentnosti.

Nova verzija mora imati:

- centralizovan servis za dodelu delovodnog broja,
- atomic ili optimistic locking mehanizam,
- retry logiku,
- unique constraint gde je tehnički moguće,
- audit log,
- error log,
- kontrolisan response prema Power Apps aplikaciji,
- jasnu poruku korisniku ako broj nije moguće rezervisati.

---

## 5. Trenutno poznate komponente

### Činjenice

Na osnovu do sada dostavljenih XML exporta i dokumenata, poznato je sledeće:

- Rešenje koristi SharePoint Online site `DocumentCentralv6.0`.
- Postoji lista `Svi predmeti`.
- Postoji lista `Partneri`.
- Postoji lista `AppConfig`.
- Postoji lista `RezervisaniBrojevi`.
- Postoji biblioteka `Shared Documents`.
- Postoji biblioteka `EmailDocuments`.
- Postoji biblioteka `Exports`.
- `Svi predmeti` je centralna evidencija predmeta/dokumenata.
- `Partneri` je evidencija poslovnih partnera/kontakata.
- `Shared Documents` sadrži dokumente povezane sa delovodnim brojem.
- `EmailDocuments` najverovatnije služi za dokumente dobijene iz email procesa.
- `Exports` najverovatnije služi za exportovane fajlove i izveštaje.
- `AppConfig` ima višelinijsko tekstualno polje `Config`.
- `AppConfig` export trenutno sadrži 22 konfiguracione stavke.
- `AppConfig.Config` sadrži JSON konfiguraciju za šifarnike, procese, prevode, korisnike, permisije, AppLock i SEF/DC datum povlačenja.
- `RezervisaniBrojevi` ima numeričko polje `RezervisaniBroj`.

### Pretpostavke

Sledeće stavke su logične pretpostavke, ali moraju biti potvrđene kroz solution, flow definicije ili dodatne exporte:

- `RezervisaniBrojevi` se koristi za rezervaciju ili kontrolu delovodnih brojeva.
- `EmailDocuments` je deo email intake procesa.
- `Exports` se koristi za generisane export fajlove.
- Power Apps aplikacija koristi SharePoint liste za čitanje i/ili pisanje podataka.
- Deo backend logike je implementiran kroz Power Automate flow-ove.

### Nepoznato

Trenutno nije potvrđeno:

- kompletna struktura Canvas aplikacije,
- tačni ekrani aplikacije,
- sve kolekcije u aplikaciji,
- sve formule i OnSelect/OnVisible/OnStart logike,
- svi Power Automate flow-ovi,
- triggeri flow-ova,
- flow koji generiše delovodni broj,
- da li postoji ETag / optimistic concurrency,
- da li postoji unique constraint nad finalnim delovodnim brojem,
- da li postoji centralni audit log,
- da li postoji error log,
- da li korisnici pišu direktno u SharePoint ili isključivo preko flow-ova,
- security model i item-level permissions logika.

---

## 6. Identifikovane SharePoint liste i biblioteke

| Naziv | Tip | Poznata namena | Status |
|---|---|---|---|
| `Svi predmeti` | SharePoint lista | Centralna evidencija predmeta/dokumenata | Potvrđeno iz XML exporta |
| `Partneri` | SharePoint lista | Evidencija partnera/kontakata | Potvrđeno iz XML exporta |
| `AppConfig` | SharePoint lista | Centralna konfiguracija, šifarnici, procesi, permisije, prevodi i sistemska podešavanja | Potvrđeno iz CSV exporta |
| `RezervisaniBrojevi` | SharePoint lista | Rezervacija/kontrola brojeva | Pretpostavka na osnovu naziva i polja |
| `Shared Documents` | Document library | Glavna biblioteka dokumenata | Delimično potvrđeno |
| `EmailDocuments` | Document library | Email dokumenti | Pretpostavka |
| `Exports` | Document library | Exporti/izveštaji | Pretpostavka |

---

## 7. Data model dokumenti

Detaljna dokumentacija SharePoint lista se vodi u folderu `data-model`.

Trenutno pripremljeni fajlovi:

```text
data-model/
├── svi-predmeti.md
├── partneri.md
└── app-config/
    ├── README.md
    ├── delovodneknjige.md
    ├── procesconfig.md
    ├── stanja.md
    ├── permisije.md
    ├── translations.md
    ├── users.md
    ├── arhivskaknjigagrupekategorija.md
    ├── arhivskalistakategorije.md
    ├── kategorijedokumentarnogmaterijala.md
    ├── organizacionejedinice.md
    ├── lokacijeregistracionihjedinica.md
    ├── rokovicuvanja.md
    ├── tipovidokumenta.md
    ├── tipregistracionejedinice.md
    ├── valute.md
    ├── vrstedokumenta.md
    ├── registracionejedinice.md
    ├── efaktureparametri.md
    ├── settings.md
    ├── applock.md
    └── sefdcdatumpovlacenja.md
```

### 7.1 `Svi predmeti`

Fajl:

```text
data-model/svi-predmeti.md
```

Status iz dokumentacije:

- lista ima 126 kolona u dostavljenom exportu,
- 65 kolona je vidljivo,
- 61 kolona je skrivena,
- nema kolone sa `Required = true` u exportu,
- nema kolone sa `EnforceUniqueValues = true` u exportu,
- `DelovodniBroj` nije označen kao unique,
- `DelovodniBroj` nije indeksiran u exportu,
- `EDokument` je Boolean kolona sa default vrednošću `0`,
- `EdocStatus` je Choice kolona sa default vrednošću `New`,
- `LinkDoDokumenta` ima JSON column formatting za dugme/link „Pregledaj dokument”.

Najvažniji rizik:

```text
DelovodniBroj nije unique na nivou SharePoint liste.
```

To znači da enterprise garancija jedinstvenog delovodnog broja mora biti rešena backend logikom, ne samo UI logikom.

### 7.2 `Partneri`

Fajl:

```text
data-model/partneri.md
```

Status iz dokumentacije:

- lista ima 94 kolone u dostavljenom exportu,
- poslovno upisive vidljive kolone uključuju `Title`, `Adresa`, `Grad`, `TipKontakta`, `PIB`, `JMBG`, `VrstaKontakta`, `MaticniBroj`, `BCID`, `Klasifikacija`, `Attachments`,
- `Title` je jedina poslovna kolona označena kao `Required = true`,
- nema kolone sa `EnforceUniqueValues = true`,
- nema indeksirane poslovne kolone,
- `TipKontakta` je Choice kolona, ali bez definisanih choice vrednosti u exportu,
- `PIB`, `JMBG`, `MaticniBroj`, `BCID` i `Klasifikacija` su tekstualne kolone.

Najvažniji rizik:

```text
PIB i BCID nisu unique i nisu indeksirani.
```

Ako integracija proverava partnera po PIB-u, preporuka je da `PIB` bude indeksiran i da se definiše kontrolisano pravilo jedinstvenosti.

### 7.3 `AppConfig`

Folder:

```text
data-model/app-config/
```

Status iz dokumentacije:

- `AppConfig` je centralna konfiguraciona lista aplikacije.
- Export sadrži 22 konfiguracione stavke.
- Kolona `Config` sadrži JSON nizove ili JSON vrednosti.
- Za svaku konfiguracionu stavku napravljen je poseban `.md` dokument.
- Sirovi JSON sadržaji su izdvojeni u folder `evidence/app-config-json/`.

Dokumentovane konfiguracione oblasti:

| Konfiguracija | Namena |
|---|---|
| `DelovodneKnjige` | Definicije delovodnih knjiga / knjiga evidentiranja |
| `ProcesConfig` | Konfiguracija procesa, koraka i logike toka |
| `Stanja` | Statusi/stanja dokumenata ili procesa |
| `Permisije` | Konfiguracija permission modela na aplikativnom nivou |
| `Translations` | Prevodi i lokalizacija aplikacije |
| `Users` | Korisnici ili sistemski definisani korisnički zapisi |
| `ArhivskaKnjigaGrupeKategorija` | Grupe kategorija arhivske knjige |
| `ArhivskaListaKategorije` | Kategorije arhivske liste |
| `KategorijeDokumentarnogMaterijala` | Kategorije dokumentarnog materijala |
| `OrganizacioneJedinice` | Organizacione jedinice |
| `LokacijeRegistracionihJedinica` | Lokacije registracionih jedinica |
| `RokoviCuvanja` | Rokovi čuvanja dokumentacije |
| `TipoviDokumenta` | Tipovi dokumenata |
| `TipRegistracioneJedinice` | Tipovi registracionih jedinica |
| `Valute` | Šifarnik valuta |
| `VrsteDokumenta` | Vrste dokumenata |
| `RegistracioneJedinice` | Registracione jedinice |
| `EFaktureParametri` | Parametri za e-fakture / SEF logiku |
| `Settings` | Opšta podešavanja aplikacije |
| `AppLock` | Lock status aplikacije |
| `SEFDCdatumPovlacenja` | Datum povlačenja iz SEF/DC procesa |

Najvažniji zaključak:

```text
AppConfig nije samo tehnička konfiguracija, već nosi značajan deo poslovnih pravila aplikacije.
```

Zato se u novoj verziji mora tretirati kao deo poslovnog data modela, ne samo kao pomoćna lista.

Rizici i nepoznato:

- Nije potvrđeno koje konfiguracije se učitavaju na `App.OnStart`, `Screen.OnVisible` ili kroz flow.
- Nije potvrđeno da li aplikacija kešira `AppConfig` vrednosti lokalno u kolekcije.
- Nije potvrđena validacija JSON strukture pre upisa u `Config`.
- Nije potvrđeno da li postoji versioning konfiguracije.
- Nije potvrđeno da li postoji audit promena konfiguracije.
- Nije potvrđeno ko ima pravo izmene `AppConfig` liste.

Enterprise preporuke za novu verziju:

- Definisati JSON schema za svaku konfiguracionu oblast.
- Uvesti versioning konfiguracije.
- Uvesti audit log za izmene konfiguracije.
- Uvesti jasnu podelu na sistemsku konfiguraciju, šifarnike i poslovna pravila.
- Kritične konfiguracije validirati backend logikom pre primene.
- Razmotriti posebne liste za velike šifarnike ako `AppConfig` JSON postane prevelik za održavanje.

---

## 8. Visok nivo trenutne arhitekture

```text
Power Apps Canvas App
        |
        | korisnik unosi, pregleda i pokreće procese
        v
Power Automate flows
        |
        | validacija, kreiranje, ažuriranje, integracije, dokumenti
        v
SharePoint Online
        |
        | liste + biblioteke + metadata + permissions
        v
DocCentral dokumenti, predmeti, partneri, konfiguracija i exporti
```

---

## 9. Ciljna enterprise arhitektura

Nova verzija treba da ide ka arhitekturi u kojoj je Canvas aplikacija tanak i responzivan UI, a kritična poslovna logika se izvršava kroz kontrolisane backend servise.

### Canvas aplikacija treba da radi

- prikaz podataka,
- validaciju obaveznih polja,
- navigaciju,
- korisnički unos,
- pozivanje Power Automate flow-ova,
- prikaz statusa i grešaka,
- višejezičnost: srpski i engleski,
- responzivan layout.

### Canvas aplikacija ne treba da radi sama

- finalno generisanje delovodnog broja,
- garanciju jedinstvenosti broja,
- kritične upise bez server-side validacije,
- permission logiku koja mora biti enforce-ovana na backendu,
- kompleksne integracione transformacije bez flow/backend sloja.

### Power Automate / backend treba da radi

- Create/Edit/Delete operacije,
- dodelu delovodnog broja,
- optimistic locking,
- retry,
- audit log,
- error log,
- integracije sa SAP/Business Central/SEF ako postoje,
- validaciju pre upisa,
- kontrolisano ažuriranje SharePoint podataka.

---

## 10. Security i permission pravila za novu verziju

Za novu verziju važi sledeći cilj:

- korisnici imaju `Read Only` prava nad SharePoint listama i bibliotekama,
- sve Create/Edit/Delete aktivnosti idu preko Power Automate flow-ova,
- Power Automate koristi servisni nalog ili connection reference sa odgovarajućim pravima,
- korisnik ne sme moći da zaobiđe aplikaciju i direktno menja kritične SharePoint podatke,
- svaki kritičan upis mora imati audit trag.

Ovo pravilo je posebno važno za:

- `Svi predmeti`,
- dodelu delovodnog broja,
- izmene statusa,
- rad sa dokumentima,
- izmene partnera iz integracije.

---

## 11. Pravila za dokumentaciju

Svi fajlovi u ovom repozitorijumu moraju jasno razdvajati:

- činjenice,
- pretpostavke,
- nepoznato,
- rizike,
- preporuke.

Ne sme se izmišljati logika koja nije potvrđena iz:

- Power Platform solution fajla,
- SharePoint XML metadata exporta,
- Power Apps formula,
- Power Automate definicija,
- korisničkog objašnjenja,
- stvarne konfiguracije liste ili biblioteke.

Ako nešto nije potvrđeno, koristi se oznaka:

```text
NEPOZNATO
```

Ako je zaključak izveden iz naziva liste, naziva kolone ili konteksta, mora biti označen kao:

```text
PRETPOSTAVKA
```

Ako je potvrđeno direktno iz exporta ili solution fajla, označava se kao:

```text
ČINJENICA
```

---

## 12. Predložena struktura repozitorijuma

```text
doccentral-v6-documentation/
│
├── README.md
│
├── data-model/
│   ├── svi-predmeti.md
│   ├── partneri.md
│   ├── app-config/
│   │   ├── README.md
│   │   ├── delovodneknjige.md
│   │   ├── procesconfig.md
│   │   ├── stanja.md
│   │   ├── permisije.md
│   │   └── ...
│   ├── rezervisani-brojevi.md
│   ├── shared-documents.md
│   ├── email-documents.md
│   └── exports.md
│
├── docs/
│   ├── 01-business-overview.md
│   ├── 02-current-architecture.md
│   ├── 03-target-architecture.md
│   ├── 04-sharepoint-data-model.md
│   ├── 05-power-apps-analysis.md
│   ├── 06-power-automate-analysis.md
│   ├── 07-security-permissions.md
│   ├── 08-numbering-and-concurrency.md
│   ├── 09-integrations.md
│   ├── 10-known-issues-technical-debt.md
│   ├── 11-enterprise-improvements.md
│   └── 12-cloud-code-development-brief.md
│
├── architecture/
│   ├── current-architecture.md
│   ├── target-architecture.md
│   ├── numbering-sequence.md
│   ├── security-model.md
│   └── integration-model.md
│
├── claude-code/
│   ├── README.md
│   ├── 01-project-context.md
│   ├── 02-claude-code-system-instructions.md
│   ├── 03-analysis-workflow.md
│   ├── 04-cloud-code-development-brief.md
│   └── 05-naming-conventions.md
│
├── templates/
│   ├── 01-business-overview-template.md
│   ├── 02-architecture-template.md
│   ├── 03-sharepoint-list-template.md
│   ├── 04-power-apps-screen-template.md
│   ├── 05-power-automate-flow-template.md
│   ├── 06-security-model-template.md
│   ├── 07-known-issues-template.md
│   └── 08-cloud-code-prompt-template.md
│
├── checklists/
│   ├── 01-solution-analysis-checklist.md
│   ├── 02-sharepoint-data-model-checklist.md
│   ├── 03-power-apps-checklist.md
│   ├── 04-power-automate-checklist.md
│   ├── 05-security-permissions-checklist.md
│   ├── 06-concurrency-checklist.md
│   ├── 07-deployment-checklist.md
│   └── 08-documentation-quality-checklist.md
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

## 13. Claude Code / Cloud Code pravila

Claude Code instrukcije i skillovi treba da budu smešteni u folder:

```text
claude-code/
```

Ovaj folder treba da sadrži pravila za AI-assisted development.

### Obavezna pravila za Claude Code

Claude Code mora:

- da koristi postojeću dokumentaciju kao izvor istine,
- da ne izmišlja SharePoint kolone, liste, flow-ove ili Power Apps kontrole,
- da svaku nepoznatu stvar označi kao `NEPOZNATO`,
- da razdvaja činjenice od pretpostavki,
- da predlaže enterprise rešenja, ne samo brza UI rešenja,
- da obrati posebnu pažnju na delovodni broj i concurrency,
- da koristi postojeće internal names SharePoint kolona,
- da ne menja internal names bez migracionog plana,
- da predlaže responzivan Canvas app,
- da predlaže srpski i engleski jezik u aplikaciji,
- da Create/Edit/Delete operacije vodi kroz Power Automate,
- da korisnike tretira kao Read Only na SharePoint data layeru.

---

## 14. Checkliste

Folder:

```text
checklists/
```

Checkliste služe za kontrolu kvaliteta analize i razvoja.

Obavezne oblasti:

- solution analiza,
- SharePoint data model,
- Power Apps analiza,
- Power Automate analiza,
- security/permissions,
- concurrency,
- deployment,
- kvalitet dokumentacije.

---

## 15. Templates

Folder:

```text
templates/
```

Templates služe za standardizovano pisanje dokumentacije.

Svaki novi dokument treba da ima najmanje sledeće sekcije:

- status dokumenta,
- izvor podataka,
- činjenice,
- pretpostavke,
- nepoznato,
- rizici,
- preporuke,
- otvorena pitanja.

---

## 16. Prioriteti za dalji rad

1. Dodati dokumentaciju za `RezervisaniBrojevi`.
2. Dodati dokumentaciju za `Shared Documents`.
3. Dodati dokumentaciju za `EmailDocuments`.
4. Dodati dokumentaciju za `Exports`.
5. Analizirati Power Platform solution ZIP kada bude dostupan.
6. Ekstraktovati Power Apps Canvas app strukturu.
7. Ekstraktovati sve Power Automate flow definicije.
8. Dokumentovati sve connection references.
9. Dokumentovati environment variables.
10. Dokumentovati security/permission model.
11. Mapirati gde se svaka `AppConfig` konfiguracija koristi u Canvas aplikaciji i flow-ovima.
12. Definisati target enterprise arhitekturu.
13. Napisati finalni Cloud Code master prompt za novu verziju.

---

## 17. Known issues / tehnički dug

Trenutno poznati ili verovatni rizici:

### 17.1 Delovodni broj

- `DelovodniBroj` u listi `Svi predmeti` nije potvrđen kao unique.
- `DelovodniBroj` nije potvrđen kao indeksiran.
- Nije potvrđeno da postoji atomic/optimistic locking.
- Nije potvrđeno da postoji retry logika.
- Nije potvrđeno da postoji audit log za dodelu broja.

### 17.2 Partneri

- `PIB` nije potvrđen kao unique.
- `PIB` nije potvrđen kao indeksiran.
- `BCID` nije potvrđen kao unique.
- `BCID` nije potvrđen kao indeksiran.
- Nije potvrđen primarni ključ za sinhronizaciju partnera.
- Nije potvrđena logika ažuriranja adrese, grada i naziva partnera iz SAP/BC integracije.

### 17.3 AppConfig

- `AppConfig` sadrži veliki deo poslovne konfiguracije u JSON formatu.
- Nije potvrđena JSON schema validacija.
- Nije potvrđen audit promena konfiguracije.
- Nije potvrđeno ko ima pravo izmene konfiguracionih stavki.
- Nije potvrđeno koje konfiguracije su kritične za delovodni broj, procese, permisije i arhivsku knjigu.

### 17.4 Power Apps

- Struktura ekrana trenutno nije dokumentovana.
- Kolekcije nisu kompletno dokumentovane.
- Nije potvrđeno koje liste aplikacija direktno piše.
- Nije potvrđeno koje operacije idu preko flow-ova.

### 17.5 Power Automate

- Flow definicije nisu analizirane u ovom README fajlu.
- Nisu dokumentovani triggeri.
- Nisu dokumentovane konekcije.
- Nisu dokumentovane connection references.
- Nisu dokumentovani error handling i retry mehanizmi.

---

## 18. Enterprise preporuke

### 18.1 Brojevi i zavodna knjiga

- Uvesti centralni flow ili backend servis za generisanje delovodnog broja.
- Koristiti SharePoint ETag ili drugi optimistic locking pristup.
- Dodati retry do unapred definisanog broja pokušaja.
- Dodati listu za audit dodele brojeva.
- Dodati listu za error log.
- Dodati kontrolu jedinstvenosti broja.
- Razdvojiti rezervaciju broja od finalnog upisa dokumenta.

### 18.2 Partneri

- Definisati primarni ključ partnera.
- Kandidati: `PIB`, `BCID`, `MaticniBroj` ili kombinacija.
- Indeksirati kolone koje se koriste u filterima.
- Definisati pravilo automatskog update-a partnera iz SAP/BC izvora.
- Logovati staru i novu vrednost kod integracionih izmena.

### 18.3 AppConfig

- Podeliti konfiguraciju na logičke oblasti: šifarnici, procesna pravila, permission pravila, UI/localization podešavanja i sistemska podešavanja.
- Definisati JSON schema za svaku `AppConfig` stavku.
- Uvesti kontrolisanu izmenu konfiguracije kroz aplikaciju ili administrativni flow.
- Uvesti audit log za svaku izmenu konfiguracije.
- Uvesti validaciju pre primene konfiguracije u aplikaciji.
- Dokumentovati mapiranje: `AppConfig` stavka → kolekcija u Canvas app → ekran/proces koji je koristi.

### 18.4 SharePoint

- Indeksirati kolone koje se koriste za filtere.
- Proveriti list view threshold rizike.
- Proveriti unique constraints tamo gde su potrebni.
- Proveriti da li postoje direktni korisnički upisi.
- Proveriti permissions model za liste i biblioteke.

### 18.5 Power Apps

- Smanjiti poslovnu logiku u Canvas aplikaciji.
- Koristiti responzivne layout container-e.
- Uvesti centralizovanu validaciju.
- Uvesti lokalizaciju srpski/engleski.
- Uvesti standardizovane komponente.

### 18.6 Power Automate

- Standardizovati error handling.
- Standardizovati logging.
- Koristiti child flow-ove za ponavljajuće operacije.
- Koristiti connection references.
- Dokumentovati sve trigger condition-e.
- Dokumentovati sve Parse JSON sheme.

---

## 19. GitHub radni tok

### Dodavanje novog fajla

```bash
git add <putanja-do-fajla>
git commit -m "docs: add <opis-fajla>"
git push
```

### Dodavanje celog foldera

```bash
git add data-model/
git commit -m "docs: add SharePoint data model documentation"
git push
```

### Ažuriranje README fajla

```bash
git add README.md
git commit -m "docs: update project README"
git push
```

---

## 20. Minimalni standard za svaki dokument

Svaki `.md` fajl treba da ima:

```text
# Naslov

## Status dokumenta

## Izvor podataka

## Činjenice

## Pretpostavke

## Nepoznato

## Rizici

## Preporuke

## Otvorena pitanja
```

---

## 21. Najvažniji zaključak

DocCentral v6.0 se mora dalje dokumentovati i razvijati kao enterprise sistem za evidenciju dokumenata, a ne kao jednostavna Canvas aplikacija nad SharePoint listama.

Najkritičniji deo sistema je:

```text
concurrency-safe generisanje i čuvanje jedinstvenog delovodnog broja
```

Sve buduće analize, dokumentacija, Claude Code instrukcije i razvoj nove verzije moraju tretirati ovaj zahtev kao obavezan enterprise standard.
