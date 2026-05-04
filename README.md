# DocCentral v6.0

## Tehnička dokumentacija, pravila analize i Claude Code razvojna osnova

Ovaj repozitorijum sadrži dokumentaciju i razvojna uputstva za **DocCentral v6.0**, Power Platform rešenje za elektronsku pisarnicu, evidenciju dokumenata i rad sa SharePoint listama/bibliotekama.

Repozitorijum je namenjen za:

- tehničku dokumentaciju,
- SharePoint data model dokumentaciju,
- Power Apps analizu,
- Power Automate analizu,
- Claude Code / Cloud Code skillove,
- šablone,
- checkliste,
- razvojni brief za novu enterprise verziju.

---

## 1. Svrha repozitorijuma

Cilj repozitorijuma je da bude centralno mesto za razumevanje postojećeg DocCentral rešenja i pripremu nove unapređene verzije.

Dokumentacija mora da omogući:

1. Razumevanje poslovne svrhe aplikacije.
2. Razumevanje trenutne Power Platform arhitekture.
3. Dokumentovanje SharePoint data modela.
4. Dokumentovanje Power Apps Canvas aplikacije.
5. Dokumentovanje Power Automate flow-ova.
6. Identifikaciju rizika i tehničkog duga.
7. Definisanje enterprise unapređenja.
8. Pripremu jasnog inputa za Claude Code razvoj.

---

## 2. Osnovni opis rešenja

**DocCentral v6.0** je Power Platform rešenje za elektronsku pisarnicu i centralnu evidenciju dokumenata.

Rešenje koristi:

- Power Apps Canvas aplikaciju kao UI,
- Power Automate flow-ove za backend logiku,
- SharePoint Online liste kao data layer,
- SharePoint Online biblioteke kao document storage,
- App Config listu za šifarnike i konfiguracije,
- posebne flow-ove i listu za rezervaciju/kontrolu delovodnih brojeva.

---

## 3. SharePoint okruženje

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

REST API base:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/
```

---

## 4. Ključni enterprise zahtev

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

Nova verzija mora imati:

- centralizovan Numbering Service,
- atomic ili optimistic locking,
- retry logiku,
- unique constraint gde je moguće,
- audit log,
- error log,
- correlation/operation ID,
- kontrolisan response prema Power Apps aplikaciji.

---

## 5. Činjenice iz postojeće dokumentacije

Potvrđeno je:

- SharePoint site `DocumentCentralv6.0`,
- lista `Svi predmeti`,
- lista `Partneri`,
- lista `AppConfig`,
- lista `RezervisaniBrojevi`,
- biblioteka `Shared Documents`,
- biblioteka `EmailDocuments`,
- biblioteka `Exports`,
- `AppConfig.Config` sadrži konfiguracije/šifarnike,
- `RezervisaniBrojevi` postoji kao deo numbering modela,
- korisnici treba da imaju SharePoint Read Only,
- upisi treba da idu preko Power Automate pod service account-om,
- pristup dokumentima je zasnovan na organizacionim jedinicama/grupama,
- backend permissions moraju sakriti iteme/fajlove, ne samo UI.

---

## 6. Činjenice iz Power Platform solution-a

| Oblast | Nalaz |
|---|---|
| Canvas aplikacija | 1 |
| Canvas ekrani | 16 |
| Power Automate flow-ovi | 20 |
| Environment variables | 9 |
| Connection references | 5 |
| App version | `g_varVersion = 2.1.155` |
| Current user | `g_varCurrentUser = Office365Users.MyProfileV2()` |
| Master config | `colMasterData` iz `EV_DocCentral21_lstAppConfig` |

---

## 7. Identifikovani Canvas ekrani

- `scrHome`
- `scrAddDocument`
- `scrAddReminders`
- `scrReserveNumber`
- `scrDocSwap`
- `scrArchive`
- `scrCodes`
- `scrArchiveBook`
- `scrArchiveBookItems`
- `scrLockYear`
- `scrSettings`
- `scrPartneri`
- `scrEDocuments`
- `scrEmailDocs`
- `scrEmail`
- `scrProcess`

Detalji:

```text
power-apps/screens.md
power-apps/app-onstart.md
power-apps/collections.md
power-apps/navigation.md
docs/05-power-apps-analysis.md
```

---

## 8. Identifikovani Power Automate flow-ovi

- `CF_DocCentral21_AddPartner`
- `CF_DocCentral21_UploadDoc_V4`
- `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- `CF_DocCentral21_GetPartners`
- `CF_DocCentral21_ResetDelovodnika`
- `CF_DocCentral21_ProcessFlow`
- `CF_DocCentral21_EFaktureProces`
- `CF_DocCentral21_EmailDoc`
- `CF_DocCentral21_GetFileEmail`
- `CF_DocCentral21_AdHocApproval`
- `CF_DocCentral_HTMLTabela`
- `CF_DocCentral21_LockYear`
- `CF_DocCentral21_GetLockYearData`
- `CF_DocCentral21_ExportCodes`
- `CF_DocCentral21_ArchiveDocumentsV2`
- `CF_DocCentral21_CheckArchiveBookData`
- `CF_DocCentral21_DocSwap`
- `CF_DocCentral21_ReserveNumber`
- `CF_DocCentral21_UploadDoc_V2`
- `CF_DocCentral21_AddReminder`

Detalji:

```text
power-automate/flow-inventory.md
docs/06-power-automate-analysis.md
```

---

## 9. Environment variables

- `pbmlv2_EV_DocCentral21_SharePointSite`
- `pbmlv2_EV_DocCentral21_lstPodestinici`
- `pbmlv2_EV_DocCentral21_lstDocuments`
- `pbmlv2_EV_DocCentral21_lstRezervisanibrojevi`
- `pbmlv2_EV_DocCentral21_lstSviPredmeti`
- `pbmlv2_EV_DocCentral21_Export`
- `gpdoccen_EV_DocCentral21_EmailDocuments`
- `gpdoccen_EV_DocCentral21_lstPartneri`
- `pbmlv2_EV_DocCentral21_lstAppConfig`

Detalji:

```text
configuration/environment-variables.md
```

---

## 10. Connection references

- `gpdoccen_CF_DocumentCentral_SharedMailbox`
- `new_sharedexcelonlinebusiness_9b86f`
- `new_sharedonedriveforbusiness_49131`
- `pbmlv2_CR_CR_DocCentral21_Office365Outlook`
- `pbmlv2_CR_DocCentral21_SharePoint`

Detalji:

```text
power-automate/connection-references.md
```

---

## 11. Struktura repozitorijuma

```text
doccentral-v6-documentation/
├── README.md
├── docs/
├── architecture/
├── data-model/
├── power-apps/
├── power-automate/
├── configuration/
├── known-issues/
├── claude-code/
├── templates/
├── checklists/
├── prompts/
└── backlog/
```

---

## 12. Pravila dokumentacije

Koristiti oznake:

```text
ČINJENICA
PRETPOSTAVKA
NEPOZNATO
PREPORUKA
```

Ne izmišljati postojeću logiku, liste, kolone, flow-ove, ekrane ili role.

---

## 13. GitHub komanda za upload

```bash
git add README.md docs/ architecture/ data-model/ power-apps/ power-automate/ configuration/ known-issues/ claude-code/ templates/ checklists/ prompts/ backlog/
git commit -m "docs: update DocCentral documentation from solution analysis"
git push
```

---

## 14. Zaključak

DocCentral v6.0 treba razvijati kao enterprise sistem za evidenciju dokumenata, a ne kao jednostavnu Canvas aplikaciju nad SharePoint listama.

Najkritičniji deo sistema ostaje:

```text
concurrency-safe generisanje i čuvanje jedinstvenog delovodnog broja
```
