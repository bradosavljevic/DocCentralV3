# Cloud flow inventory

## Lista flow-ova

| Flow | Status | Namena |
|---|---|---|
| CF_DocCentralV3_CreateDocument | Kreiran/predviđen | Centralni flow za zavođenje dokumenta |
| CF_DocCentralV3_GenerateRegistryNumber | Kreiran/predviđen | Generisanje sledećeg delovodnog broja |
| CF_DocCentralV3_UseReservedNumber | Kreiran/predviđen | Korišćenje rezervisanog broja |
| CF_DocCentralV3_AssignPermissions | Kreiran/predviđen | Dodela prava |
| CF_DocCentralV3_SendForApproval | Kreiran/predviđen | Slanje na odobrenje |
| CF_DocCentralV3_ProcessApprovalResponse | Kreiran/predviđen | Obrada odgovora |
| CF_DocCentralV3_SendReminders | Kreiran/predviđen | Slanje podsetnika |
| CF_DocCentralV3_ArchiveDocument | Kreiran/predviđen | Arhiviranje dokumenta |
| CF_DocCentralV3_CloseRegistryYear | Kreiran/predviđen | Zaključenje godine |
| CF_DocCentralV3_ExportAppConfig | Kreiran/predviđen | Export App Config |
| CF_DocCentralV3_GenerateArchiveBookPdf | Kreiran/predviđen | Generisanje PDF arhivske knjige |
| CF_DocCentralV3_LogEvent | Kreiran/predviđen | Audit logging |

## Uklonjeno / ne koristi se

```text
CF_DocCentralV3_CreateDocumentFolder
```

Razlog: dokumenti se ne kreiraju u folderima. Svi fajlovi su u root-u biblioteke.

## Centralni orkestracioni flow

```text
CF_DocCentralV3_CreateDocument
```

Ovaj flow treba da obuhvati:

- validaciju
- generisanje/korišćenje broja
- kreiranje itema
- kreiranje fajla u root biblioteci
- kreiranje priloga u root biblioteci
- anti-overwrite logiku
- dodelu prava
- ažuriranje brojača
- brisanje iskorišćenog rezervisanog broja
- audit log
- response Canvas aplikaciji
