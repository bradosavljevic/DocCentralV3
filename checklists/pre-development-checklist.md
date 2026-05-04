# Checklist: pre razvoja

## Solution

- [ ] Solution se zove `DocCentralV3`
- [ ] Publisher je `GoProDocCentral`
- [ ] Version je `3.0.0.0`
- [ ] Canvas app `DocCentralV3` postoji u solution-u
- [ ] Svi cloud flow-ovi su dodati u solution
- [ ] Sve environment variables su dodate u solution
- [ ] Sve connection references su dodate u solution

## Environment variables

- [ ] `EV_DocCentralV3_SharePointSite`
- [ ] `EV_DocCentralV3_lstSviPredmeti`
- [ ] `EV_DocCentralV3_lstPartneri`
- [ ] `EV_DocCentralV3_lstAppConfig`
- [ ] `EV_DocCentralV3_lstPodsetnici`
- [ ] `EV_DocCentralV3_lstRezervisaniBrojevi`
- [ ] `EV_DocCentralV3_lstAuditLog`
- [ ] `EV_DocCentralV3_docDokumenti`
- [ ] `EV_DocCentralV3_docEmailDocs`
- [ ] `EV_DocCentralV3_docExports`

## Connection references

- [ ] `CR_DocCentralV3_SharePoint`
- [ ] `CR_DocCentralV3_Outlook`
- [ ] `CR_DocCentralV3_Office365Users`
- [ ] `CR_DocCentralV3_Office365Groups`
- [ ] `CR_DocCentralV3_OneDrive`
- [ ] `CR_DocCentralV3_Excel`

## SharePoint

- [ ] Lista `Svi predmeti`
- [ ] Lista `Partneri`
- [ ] Lista `App Config`
- [ ] Lista `Podsetnici`
- [ ] Lista `Rezervisani brojevi`
- [ ] Lista `AuditLog`
- [ ] Biblioteka `Dokumenti`
- [ ] Biblioteka `EmailDocs`
- [ ] Biblioteka `Exports`

## Važne odluke

- [ ] Dokumenti se ne čuvaju u folderima
- [ ] Svi fajlovi idu u root biblioteke
- [ ] Anti-overwrite pravilo je dokumentovano
- [ ] `CF_DocCentralV3_CreateDocumentFolder` se ne koristi
- [ ] Sav Claude Code output ide u `PACode`
