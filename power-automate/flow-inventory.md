# Power Automate flow inventory - DocCentral V3

## Status

Flow-ovi će biti kreirani kasnije. Dokumentacija mora koristiti sledeće nazive jer su oni zaključani kao naming standard za solution `DocCentralV3`.

Svi flow-ovi koriste prefix `CF_`.

## Connection references koje flow-ovi koriste

- `CR_DocCentralV3_SharePoint`
- `CR_DocCentralV3_Outlook`
- `CR_DocCentralV3_Office365Users`
- `CR_DocCentralV3_Office365Groups`
- `CR_DocCentralV3_OneDrive`
- `CR_DocCentralV3_Excel`

## Environment variables koje flow-ovi koriste

- `EV_DocCentralV3_SharePointSite`
- `EV_DocCentralV3_lstSviPredmeti`
- `EV_DocCentralV3_lstPartneri`
- `EV_DocCentralV3_lstAppConfig`
- `EV_DocCentralV3_lstRezervisaniBrojevi`
- `EV_DocCentralV3_lstPodsetnici`
- `EV_DocCentralV3_lstAuditLog`
- `EV_DocCentralV3_docDokumenti`
- `EV_DocCentralV3_docEmailDocs`
- `EV_DocCentralV3_docExports`

## Obavezni cloud flow-ovi

### 1. `CF_DocCentralV3_CreateDocument`

Svrha:

- centralni flow za zavođenje dokumenta
- validacija zahteva
- poziv generisanja ili korišćenja rezervisanog delovodnog broja
- kreiranje item-a u `EV_DocCentralV3_lstSviPredmeti`
- kreiranje foldera/biblioteke kroz `CF_DocCentralV3_CreateDocumentFolder`
- dodela prava kroz `CF_DocCentralV3_AssignPermissions`
- audit kroz `CF_DocCentralV3_LogEvent`
- kontrolisan response nazad u Power Apps

### 2. `CF_DocCentralV3_GenerateRegistryNumber`

Svrha:

- concurrency-safe generisanje sledećeg delovodnog broja
- čitanje i ažuriranje delovodne knjige iz `EV_DocCentralV3_lstAppConfig`
- optimistic locking / ETag ako se implementira
- retry logika
- logovanje uspešnih i neuspešnih pokušaja

### 3. `CF_DocCentralV3_UseReservedNumber`

Svrha:

- validacija rezervisanog broja iz `EV_DocCentralV3_lstRezervisaniBrojevi`
- provera aktivne godine
- korišćenje rezervisanog broja
- brisanje rezervisanog broja tek nakon uspešnog zavođenja dokumenta
- logovanje korišćenja rezervisanog broja

### 4. `CF_DocCentralV3_CreateDocumentFolder`

Svrha:

- kreiranje foldera/lokacije za dokument u `EV_DocCentralV3_docDokumenti`
- eventualno kreiranje strukture za email dokumente u `EV_DocCentralV3_docEmailDocs`
- vraćanje URL-a/lokacije dokumenta

### 5. `CF_DocCentralV3_AssignPermissions`

Svrha:

- break inheritance na item/folder/file nivou
- dodela RW prava servisnom nalogu/owner-u
- dodela Read prava članovima/viewer-ima
- dodela prava na osnovu organizacione jedinice i konfigurisanih grupa
- logovanje grešaka kod prava

### 6. `CF_DocCentralV3_SendForApproval`

Svrha:

- pokretanje approval procesa
- slanje jednom korisniku ili grupi
- promena `Stanje` u `U odobravanju`
- sekvencijalna obrada ako ima više koraka
- audit događaj

### 7. `CF_DocCentralV3_ProcessApprovalResponse`

Svrha:

- obrada odgovora odobravača
- promena polja `Stanje`
- ako je odbijeno: status `Odbijeno`, vraćanje inicijatoru
- ako je odobreno: nastavak procesa ili finalni status prema ProcesConfig
- logovanje rezultata

### 8. `CF_DocCentralV3_SendReminders`

Svrha:

- scheduled flow u 08:00
- čitanje `EV_DocCentralV3_lstPodsetnici`
- slanje email podsetnika preko `CR_DocCentralV3_Outlook`
- sprečavanje duplog slanja istog podsetnika
- logovanje grešaka

### 9. `CF_DocCentralV3_ArchiveDocument`

Svrha:

- arhiviranje dokumenta iz statusa `Zavedeno`
- upis arhivskih znakova
- promena statusa u `Arhivirano`
- logovanje arhiviranja

### 10. `CF_DocCentralV3_CloseRegistryYear`

Svrha:

- provera da su svi dokumenti iz aktivne godine `Arhivirano`
- provera da je lista rezervisanih brojeva prazna
- zaključavanje godine
- kreiranje nove delovodne knjige za sledeću godinu u `EV_DocCentralV3_lstAppConfig`
- promena aktivne godine u App Config
- logovanje zaključavanja godine

### 11. `CF_DocCentralV3_ExportAppConfig`

Svrha:

- export jednog šifarnika u Excel
- export svih šifarnika odjednom
- čuvanje export fajlova u `EV_DocCentralV3_docExports`

### 12. `CF_DocCentralV3_GenerateArchiveBookPdf`

Svrha:

- generisanje PDF Arhivske knjige u prvoj verziji
- čuvanje PDF-a u `EV_DocCentralV3_docExports` ili dogovorenu biblioteku
- logovanje generisanja

### 13. `CF_DocCentralV3_LogEvent`

Svrha:

- centralni child flow/helper flow za upis u `EV_DocCentralV3_lstAuditLog`
- prima CorrelationId, EventType, EventStatus, Severity, FlowName, PayloadJson, ErrorMessage i tehničke detalje
- koristi ga svaki kritičan flow

## Obavezni standardi

- Svaki kritičan flow mora imati error handling.
- Svaki kritičan flow mora pisati u `EV_DocCentralV3_lstAuditLog` preko `CF_DocCentralV3_LogEvent`.
- Koristiti CorrelationId kroz ceo proces.
- Vraćati kontrolisan response u Power Apps.
- Create/Edit/Delete u SharePoint raditi pod servisnim nalogom.
- Ne koristiti stara imena sa prefixom `PA_`.
