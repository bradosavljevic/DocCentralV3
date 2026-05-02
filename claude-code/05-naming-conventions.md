# Naming Conventions

## Folderi

Koristi mala slova i crtice:

- docs
- architecture
- data-model
- power-apps
- power-automate
- security
- checklists
- templates
- claude-code

## Markdown fajlovi

Format:

`NN-naziv-dokumenta.md`

Primeri:
- `01-business-overview.md`
- `02-architecture.md`
- `03-data-model.md`
- `04-security-model.md`

## Power Automate flow-ovi

Preporučeni format:

`FLOW_<Area>_<Action>_<Object>`

Primeri:
- `FLOW_Documents_Create_Document`
- `FLOW_Documents_Update_Status`
- `FLOW_Registry_Generate_Number`
- `FLOW_Audit_Write_Log`

## Power Apps ekrani

Preporučeni format:

`scr_<Area>_<Purpose>`

Primeri:
- `scr_Documents_List`
- `scr_Documents_Edit`
- `scr_Documents_View`
- `scr_Approvals_Dashboard`

## Power Apps kolekcije

Preporučeni format:

`col<EntityName>`

Primeri:
- `colDocuments`
- `colPartners`
- `colApprovals`
- `colUserRoles`

## Power Apps globalne varijable

Preporučeni format:

`gbl<Name>`

Primeri:
- `gblCurrentUser`
- `gblUserRoles`
- `gblLanguage`
- `gblIsAdmin`

## Power Apps lokalne varijable

Preporučeni format:

`loc<Name>`

Primeri:
- `locSelectedDocument`
- `locShowSpinner`
- `locValidationMessage`

## SharePoint liste

Naziv liste treba da bude poslovno razumljiv.

Tehnički interni naziv ne menjati nakon kreiranja.

Za novu verziju koristiti prefiks ako je potreban:

`DC_<EntityName>`

Primeri:
- `DC_Documents`
- `DC_Partners`
- `DC_AuditLog`
- `DC_ErrorLog`
- `DC_RegistryCounters`
