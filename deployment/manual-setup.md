# Manualni setup

## 1. Kreirati solution

Display name:

```text
DocCentralV3
```

Name:

```text
DocCentralV3
```

Publisher:

```text
GoProDocCentral
```

Version:

```text
3.0.0.0
```

Package type:

```text
Unmanaged
```

## 2. Kreirati SharePoint site

Primer:

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0
```

## 3. Kreirati liste i biblioteke

Liste:

- Svi predmeti
- Partneri
- App Config
- Podsetnici
- Rezervisani brojevi
- AuditLog

Biblioteke:

- Dokumenti
- EmailDocs
- Exports

## 4. Kreirati environment variables

- `EV_DocCentralV3_SharePointSite`
- `EV_DocCentralV3_lstSviPredmeti`
- `EV_DocCentralV3_lstPartneri`
- `EV_DocCentralV3_lstAppConfig`
- `EV_DocCentralV3_lstPodsetnici`
- `EV_DocCentralV3_lstRezervisaniBrojevi`
- `EV_DocCentralV3_lstAuditLog`
- `EV_DocCentralV3_docDokumenti`
- `EV_DocCentralV3_docEmailDocs`
- `EV_DocCentralV3_docExports`

## 5. Kreirati connection references

- `CR_DocCentralV3_SharePoint`
- `CR_DocCentralV3_Outlook`
- `CR_DocCentralV3_Office365Users`
- `CR_DocCentralV3_Office365Groups`
- `CR_DocCentralV3_OneDrive`
- `CR_DocCentralV3_Excel`

## 6. Kreirati Canvas app

Display name:

```text
DocCentralV3
```

## 7. Kreirati cloud flow-ove

Flow-ovi mogu biti inicijalno prazni/stub flow-ovi, ali moraju biti dodati u solution da ih Canvas aplikacija i dokumentacija mogu referencirati tokom razvoja.

Flow-ovi:

- `CF_DocCentralV3_CreateDocument`
- `CF_DocCentralV3_GenerateRegistryNumber`
- `CF_DocCentralV3_UseReservedNumber`
- `CF_DocCentralV3_AssignPermissions`
- `CF_DocCentralV3_SendForApproval`
- `CF_DocCentralV3_ProcessApprovalResponse`
- `CF_DocCentralV3_SendReminders`
- `CF_DocCentralV3_ArchiveDocument`
- `CF_DocCentralV3_CloseRegistryYear`
- `CF_DocCentralV3_ExportAppConfig`
- `CF_DocCentralV3_GenerateArchiveBookPdf`
- `CF_DocCentralV3_LogEvent`

Ne kreirati:

```text
CF_DocCentralV3_CreateDocumentFolder
```

## 8. Publish all customizations

Nakon dodavanja svih komponenti:

```text
Publish all customizations
```
