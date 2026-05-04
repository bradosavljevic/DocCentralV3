# Deployment model

## Model

Deployment se radi ručnim importom solution-a po tenant-u.

## Nema migracije

Migracioni plan nije potreban.

Novi klijenti kreću iz praznog okruženja.

Postojeći klijenti već koriste postojeće liste.

## Pre deployment checklist

- [ ] Kreiran SharePoint site
- [ ] Kreirane SharePoint liste
- [ ] Kreirane biblioteke/folder strukture
- [ ] Kreiran service account
- [ ] Service account ima potrebna RW prava
- [ ] Krajnji korisnici imaju Read Only
- [ ] Importovan solution
- [ ] Podešene connection references
- [ ] Podešene environment variables
- [ ] Popunjen App Config
- [ ] Popunjeni šifarnici
- [ ] Podešen Entra group mapping
- [ ] Testiran unos dokumenta
- [ ] Testirano odobravanje
- [ ] Testirano arhiviranje
- [ ] Testirano zaključenje godine


## Kreirani objekti u development solution-u

Solution:

- `DocCentralV3`
- Publisher: `GoProDocCentral`
- Version: `3.0.0.0`

Canvas app:

- `DocCentralV3`
- Logical/name: `gpdoccen_doccentralv3_d98ba`

Connection references:

- `CR_DocCentralV3_SharePoint`
- `CR_DocCentralV3_Outlook`
- `CR_DocCentralV3_Office365Users`
- `CR_DocCentralV3_Office365Groups`
- `CR_DocCentralV3_OneDrive`
- `CR_DocCentralV3_Excel`

Environment variables:

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

Cloud flow-ovi još nisu kreirani i biće dodati kasnije pod `CF_` nazivima definisanim u `power-automate/flow-inventory.md`.
