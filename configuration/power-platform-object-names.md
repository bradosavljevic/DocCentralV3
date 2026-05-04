# Power Platform object names - DocCentral V3

## Kreirani Power Platform objekti / obavezna imena

Ova dokumentacija je usklađena sa objektima koji su već ručno kreirani u Power Platform solution-u.

### Solution

| Stavka | Vrednost |
|---|---|
| Solution display name | `DocCentralV3` |
| Solution name | `DocCentralV3` |
| Publisher | `GoProDocCentral` |
| Version | `3.0.0.0` |
| Package type | `Unmanaged` u development okruženju |

### Canvas app

| Stavka | Vrednost |
|---|---|
| Display name | `DocCentralV3` |
| Logical/name | `gpdoccen_doccentralv3_d98ba` |

### Connection references

| Display name | Logical/name | Namena |
|---|---|---|
| `CR_DocCentralV3_SharePoint` | `gpdoccen_CR_DocCentralV3_SharePoint` | SharePoint liste/biblioteke |
| `CR_DocCentralV3_Outlook` | `gpdoccen_CR_DocCentralV3_Outlook` | Email obaveštenja i podsetnici |
| `CR_DocCentralV3_Office365Users` | `gpdoccen_CR_DocCentralV3_Office365Users` | Korisnici/profili |
| `CR_DocCentralV3_Office365Groups` | `gpdoccen_CR_DocCentralV3_Office365Groups` | Entra/M365 grupe |
| `CR_DocCentralV3_OneDrive` | `gpdoccen_CR_DocCentralV3_OneDrive` | Privremeni fajlovi/Excel/PDF scenariji ako je potrebno |
| `CR_DocCentralV3_Excel` | `gpdoccen_CR_DocCentralV3_Excel` | Export u Excel |

### Environment variables

| Display name | Logical/name | Tip vrednosti | Namena |
|---|---|---|---|
| `EV_DocCentralV3_SharePointSite` | `gpdoccen_EV_DocCentralV3_SharePointSite` | SharePoint site | Root SharePoint site za rešenje |
| `EV_DocCentralV3_lstSviPredmeti` | `gpdoccen_EV_DocCentralV3_lstSviPredmeti` | SharePoint list | Glavna lista dokumenata/predmeta |
| `EV_DocCentralV3_lstPartneri` | `gpdoccen_EV_DocCentralV3_lstPartneri` | SharePoint list | Partneri |
| `EV_DocCentralV3_lstAppConfig` | `gpdoccen_EV_DocCentralV3_lstAppConfig` | SharePoint list | Šifarnici i konfiguracija |
| `EV_DocCentralV3_lstRezervisaniBrojevi` | `gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi` | SharePoint list | Rezervisani delovodni brojevi |
| `EV_DocCentralV3_lstPodsetnici` | `gpdoccen_EV_DocCentralV3_lstPodsetnici` | SharePoint list | Podsetnici |
| `EV_DocCentralV3_lstAuditLog` | `gpdoccen_EV_DocCentralV3_lstAuditLog` | SharePoint list | Audit/log lista |
| `EV_DocCentralV3_docDokumenti` | `gpdoccen_EV_DocCentralV3_docDokumenti` | SharePoint document library | Dokumenti/prilozi |
| `EV_DocCentralV3_docEmailDocs` | `gpdoccen_EV_DocCentralV3_docEmailDocs` | SharePoint document library | Email dokumenti |
| `EV_DocCentralV3_docExports` | `gpdoccen_EV_DocCentralV3_docExports` | SharePoint document library | Excel/PDF export fajlovi |

### Cloud flow nazivi

Svi flow-ovi moraju koristiti prefix `CF_`, ne `PA_`.

| Flow display name |
|---|
| `CF_DocCentralV3_CreateDocument` |
| `CF_DocCentralV3_GenerateRegistryNumber` |
| `CF_DocCentralV3_UseReservedNumber` |
| `CF_DocCentralV3_CreateDocumentFolder` |
| `CF_DocCentralV3_AssignPermissions` |
| `CF_DocCentralV3_SendForApproval` |
| `CF_DocCentralV3_ProcessApprovalResponse` |
| `CF_DocCentralV3_SendReminders` |
| `CF_DocCentralV3_ArchiveDocument` |
| `CF_DocCentralV3_CloseRegistryYear` |
| `CF_DocCentralV3_ExportAppConfig` |
| `CF_DocCentralV3_GenerateArchiveBookPdf` |
| `CF_DocCentralV3_LogEvent` |

### Pravilo

Claude Code mora koristiti ova postojeća imena i ne sme predlagati nova imena za već kreirane solution objekte, osim ako korisnik izričito traži promenu.
