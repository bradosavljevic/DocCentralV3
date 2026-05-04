# DocCentralV3 — Master Implementation Plan

## Solution identity

| Property | Value |
|---|---|
| Solution name | DocCentralV3 |
| Publisher | GoProDocCentral |
| Version | 3.0.0.0 |
| Canvas App display name | DocCentralV3 |
| Canvas App logical name | gpdoccen_doccentralv3_d98ba |

## Architecture summary

| Layer | Technology | Role |
|---|---|---|
| UI | Canvas App | User interaction only |
| Backend / write | Power Automate flows | All create/update/delete operations |
| Storage | SharePoint Online | Lists, libraries, config |
| Configuration | App Config list | All codebooks and runtime config |
| Access control | Entra ID groups | Configurable per client, never hardcoded |
| Execution identity | Service account | Runs all write operations |

## Non-negotiable rules

- All files stored in document library root — no folders.
- Every file must have a unique system-generated name. Format: `{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}`. Attachments: `{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}`.
- Original file name is preserved in the `OriginalFileName` metadata column.
- Canvas App does not write directly to SharePoint for any user-initiated operation.
- App Config is the single source of truth for all codebooks. No hardcoded client values.
- Entra group IDs and names are read from App Config, not hardcoded.
- Approval state is tracked in SharePoint. PA Approvals connector is for notifications only.
- Registry number generation is concurrency-safe (ETag/If-Match + retry).
- All flows use existing connection references (see below).
- All audit events are written by `CF_DocCentralV3_LogEvent`.
- `CF_DocCentralV3_CreateDocumentFolder` is deprecated and must not be created.
- Year closing is irreversible. No unlock mechanism may be implemented.

## Connection references

| Logical name | Connector |
|---|---|
| gpdoccen_CR_DocCentralV3_SharePoint | SharePoint |
| gpdoccen_CR_DocCentralV3_Outlook | Outlook |
| gpdoccen_CR_DocCentralV3_Office365Users | Office 365 Users |
| gpdoccen_CR_DocCentralV3_Office365Groups | Office 365 Groups |
| gpdoccen_CR_DocCentralV3_OneDrive | OneDrive |
| gpdoccen_CR_DocCentralV3_Excel | Excel |

## Environment variables

| Logical name | Purpose |
|---|---|
| gpdoccen_EV_DocCentralV3_SharePointSite | SharePoint site URL |
| gpdoccen_EV_DocCentralV3_lstSviPredmeti | Svi predmeti list name |
| gpdoccen_EV_DocCentralV3_lstPartneri | Partneri list name |
| gpdoccen_EV_DocCentralV3_lstAppConfig | App Config list name |
| gpdoccen_EV_DocCentralV3_lstPodsetnici | Podsetnici list name |
| gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi | Rezervisani brojevi list name |
| gpdoccen_EV_DocCentralV3_lstAuditLog | Audit log list name |
| gpdoccen_EV_DocCentralV3_docDokumenti | Dokumenti library name |
| gpdoccen_EV_DocCentralV3_docEmailDocs | EmailDocs library name |
| gpdoccen_EV_DocCentralV3_docExports | Exports library name |

## Document statuses (default / base)

```
Zavedeno
U odobravanju
Odobreno
Odbijeno
Arhivirano
```

Custom statuses may be defined per client via ProcesConfig in App Config. Status behavior for base statuses applies only to the default process. Flows must read active process config before applying status transitions.

## Implementation phases

### Phase 0 — Design artifacts (current batch)
Design documents for all major components. No executable artifacts.

Files:
- PACode/implementation-plan.md (this file)
- PACode/decisions/approval-mechanism.md
- PACode/decisions/registry-number-concurrency.md
- PACode/flows/CF_DocCentralV3_LogEvent-design.md
- PACode/flows/CF_DocCentralV3_GenerateRegistryNumber-design.md
- PACode/flows/CF_DocCentralV3_UseReservedNumber-design.md
- PACode/flows/CF_DocCentralV3_CreateDocument-design.md
- PACode/canvas/app-structure.md

### Phase 1 — Foundation flows (after Phase 0 approval)
Executable flow definitions for the audit and number generation layer.

Flows:
- CF_DocCentralV3_LogEvent
- CF_DocCentralV3_GenerateRegistryNumber
- CF_DocCentralV3_UseReservedNumber
- CF_DocCentralV3_AssignPermissions
- CF_DocCentralV3_CreateDocument

### Phase 2 — Approval flows (after Phase 1 approval)
Flows:
- CF_DocCentralV3_SendForApproval
- CF_DocCentralV3_ProcessApprovalResponse

### Phase 3 — Operational flows (after Phase 2 approval)
Flows:
- CF_DocCentralV3_ArchiveDocument
- CF_DocCentralV3_CloseRegistryYear
- CF_DocCentralV3_SendReminders
- CF_DocCentralV3_ExportAppConfig
- CF_DocCentralV3_GenerateArchiveBookPdf

### Phase 4 — Canvas App screens (after Phase 3 approval)
Screens:
- App shell, navigation, global variables, App Config loader
- Novi predmet (document registration)
- Dokumenti za odobrenje
- Podsetnici
- Arhiviranje
- Zaključenje godine
- Partneri
- Administracija

### Phase 5 — SharePoint column reference files (parallel or on-demand)
Reference JSON files for list/library column setup under PACode/sharepoint/.
These are implementation support files, not authoritative schema definitions.

## Open items

| Item | Status | Note |
|---|---|---|
| AppConfig.csv | MISSING from repo | Treat as gap. Use loader pattern with UNKNOWN keys. Do not guess values. |
| Svi predmeti — full column schema | PARTIAL | Documented fields used; internal column names not all confirmed |
| Partneri — full column schema | PARTIAL | No full column list with internal names |
| Podsetnici — full column schema | PARTIAL | No full column list with internal names |
| Rezervisani brojevi — full column schema | PARTIAL | No full column list with internal names |
| Approval list schema | NOT DOCUMENTED | To be defined in approval-mechanism.md |
| PDF generation design | NOT DOCUMENTED | GenerateArchiveBookPdf has no design doc yet |
| ExportAppConfig design | NOT DOCUMENTED | No design doc yet |
