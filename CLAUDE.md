# Claude Code Master Instructions

This repository describes the refactor of the existing DocCentral / e-pisarnica Power Platform solution.

Claude Code must follow these rules strictly.

## Project intent

This is a **refactor** of the existing application, not a full rebuild from scratch.

Preserve existing business behavior wherever possible.

Minimal UI redesign is allowed, but the app must become fully responsive and must support Serbian and English.

The app is primarily embedded in a SharePoint site but must also work as a standalone Canvas app on desktop, tablet and mobile devices.

## Manual implementation boundary

Claude Code works primarily on:
- Canvas app source and formulas
- Documentation
- Prompts
- Refactoring plans
- Implementation guides

Claude Code must not assume it can directly create or deploy:
- Power Automate flows
- SharePoint lists
- SharePoint libraries
- SharePoint columns
- SharePoint views
- SharePoint permissions
- environment variables
- connection references

When a new backend component is required, Claude Code must create a manual implementation guide.

The human developer will manually implement the component, test it, and return the final technical details.

Only after confirmation may Claude Code update the Canvas app to use the new component.

Claude Code must wait for:
- final flow name
- final input parameters
- final output schema
- final SharePoint list/library names
- final internal column names
- confirmation that the component was tested

## Architecture rules

- SharePoint is the data source.
- End users have Read-only permissions on SharePoint.
- All Create, Update and Delete operations must go through Power Automate.
- Power Automate must use service account connections for writes.
- Canvas app must not directly Patch, Remove, UpdateIf or SubmitForm to SharePoint data sources.
- Canvas app may Patch local collections only.
- Reading from SharePoint is allowed only where delegable and performant.
- For large datasets, use Power Automate read flows returning JSON.

## AppConfig rules

`AppConfig` is the central configuration store.

Each AppConfig item contains:
- `Title`: configuration key
- `Config`: JSON payload
- `ColumnHeader`: optional schema/header definition

Core AppConfig keys:
- `DelovodneKnjige`
- `Stanja`
- `ProcesConfig`
- `TipoviDokumenta`
- `VrsteDokumenta`
- `OrganizacioneJedinice`
- `Translations`
- `Settings`
- `RegistracioneJedinice`
- `RokoviCuvanja`
- `ArhivskaListaKategorije`
- `KategorijeDokumentarnogMaterijala`
- `EfaktureParametri`
- `Users`
- `Valute`

Rules:
- Do not hardcode dictionaries in formulas.
- Load dictionaries from AppConfig.
- Do not rename AppConfig keys without explicit approval.
- AppConfig updates must go through Power Automate.
- AppConfig updates must validate JSON schema before writing.
- Invalid schema must reject the update.
- Each JSON row should have an `ID` field as GUID where possible.
- There should be only one e-invoice config key: `EfaktureParametri`.
- Existing duplicate/legacy key `EFaktureParametri` must be treated as technical debt, not deleted automatically.

## Registry number rules

Registry number / `DelovodniBroj` is business-critical.

Rules:
1. No duplicate registry numbers are allowed.
2. Registry numbers must never be skipped.
3. Numbers must be assigned sequentially.
4. The only exception is explicitly reserved numbers.
5. Reserved numbers are valid only for the active year.
6. Reserved numbers may be used for any document type.
7. A used reserved number must be deleted from `Rezervisani brojevi` only after both `Svi predmeti` and `Dokumenta` are successfully updated.
8. If number assignment partially fails, all new document registrations must be blocked until the issue is resolved.
9. `Svi predmeti` is the source of truth for assigned registry numbers.
10. `Dokumenta` metadata must be synchronized with `Svi predmeti`.
11. Number assignment must use a lock mechanism: `NumberLocks` / `ZakljucavanjeBrojeva`.
12. Lock expires after 5 minutes.
13. Expired lock must not automatically allow sequence continuation without recovery validation.
14. If `FailedNeedsRecovery` exists, ordinary users must not continue registration, skip numbers or assign another number.
15. Only admin/consultant or service recovery flow may resolve number assignment recovery.

## Technical status rules

Business status must stay in `Stanje`.

Technical failures must not be mixed with business status.

Use separate technical process state, for example:
- `None`
- `DocumentCreationInProgress`
- `NumberAssignmentInProgress`
- `FailedNeedsRecovery`
- `MetadataSyncFailed`
- `Completed`

A `Svi predmeti` item may temporarily exist without `DelovodniBroj` only while the transaction is in progress. It must not be treated as registered until number assignment completes.

## Main document and attachments

- Main document is mandatory.
- Registration without a main document is not allowed.
- Attachments are optional.
- Current app does not support attachments at initial registration, but refactored version should support one or more attachments if feasible.
- Attachment support must not block core registration.
- Attachments are stored in `Dokumenta`.
- Attachments share the same `DelovodniBroj` as main document.
- Main document has `Attachment = false`.
- Attachments have `Attachment = true`.
- Attachment status follows the main document status.

## Approval process rules

Approval is driven by `AppConfig / ProcesConfig`.

- Process is resolved by `TipDokumenta`.
- If no active process exists for `TipDokumenta`, document is registered as `Zavedeno`.
- User does not manually choose whether document goes into process.
- Approval may be assigned to one user or one group.
- If approval is assigned to a group, all group members may be notified, but the first valid response completes the step.
- Rejection returns document to initiator.
- Initiator can edit metadata and restart process.
- Status changes through `Stanje`.
- Approval mode must be configurable:
  - `App`
  - `PowerAutomateApprovals`
- Initiator receives email when document is approved or rejected.

## Email documents

`EmailDocuments` is an intake/staging library.

- Email attachments are stored as files in `EmailDocuments`.
- No `Svi predmeti` item is created until a user processes the document.
- When processed, the file is copied to `Dokumenta`.
- Metadata is written to `Svi predmeti`.
- Processed EmailDocuments may be cleaned periodically.
- Retention should be configurable in AppConfig.
- Recommended default: 30 days after processing.

## Partner rules

- Partneri remain a separate SharePoint list.
- Partneri are not stored in AppConfig.
- `Svi predmeti` must store partner name as single line text snapshot.
- Do not use Partner lookup in `Svi predmeti`.
- Partner deletion must not affect historical documents.
- Partner deletion is hard delete, meaning item goes to SharePoint recycle bin.
- Existing documents must keep `ImePartnera`.

## Archive and year closing

- Direct transition `Zavedeno -> Arhivirano` is allowed only if document is not in process.
- After approval ends, last business status is `Zavedeno`.
- Archiving is manual after metadata is completed.
- Archive book is generated once per year.
- Duplicate archive book generation for the same year is not allowed.
- Existing archive book must not be updated or regenerated.
- Archive book has its own `DelovodniBroj`.
- Archive book uses the same registry book/counter as other documents.
- Archive book is registered in `Svi predmeti` and `Dokumenta`.
- Closing year is manual.
- Closed year can never be reopened.
- Closing year updates existing `DelovodneKnjige` JSON.
- It does not create a new `DelovodneKnjige` item.
- New active year is set.
- Counter resets to 1.
- Reserved numbers must be used before year closing.
- Reserved numbers must not be manually deleted or moved to next year.

## Security and permissions

- Users are read from Entra ID.
- Privileges and settings may be read from AppConfig.
- Future role model must be designed but not fully implemented now.
- Placeholder roles:
  - Admin
  - Consultant
  - Registrar
  - Approver
  - Archivist
  - Viewer
- Optional organization-unit permissions may be enabled.
- A user may belong to multiple organizational units.
- Process participants retain document visibility permanently.

## Canvas coding standards

Use `ai-instructions/canvas-coding-standards.md`.

## Large list loading

Some lists may exceed 2,000 items:
- Partneri
- Svi predmeti
- Dokumenta
- EmailDocuments
- documents for archiving

Claude must not rely on direct Canvas queries for large datasets if delegation/performance/threshold issues may occur.

Use Power Automate read flows that:
- use SharePoint REST API or Get items with OData filters
- select only required columns
- support pagination
- return JSON to Power Apps
- allow Power Apps to build local collections
- avoid N+1 lookup patterns

## Review process

Before changing formulas:
1. Explain current behavior.
2. Explain the problem.
3. Propose the change.
4. Confirm business behavior remains unchanged.
5. Make small changes screen-by-screen or flow-by-flow.


## Development environment links

Power Apps Studio / edit link:

```text
https://make.powerapps.com/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/canvas/?action=edit&app-id=%2Fproviders%2FMicrosoft.PowerApps%2Fapps%2Fd17711d4-1417-4b8d-ac63-df49131264d0&solution-id=a2ca7e7f-8547-f111-bec6-0022489d3551
```

Runtime / play link:

```text
https://apps.powerapps.com/play/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/a/d17711d4-1417-4b8d-ac63-df49131264d0?tenantId=b5f62db0-39d2-4763-9c01-9f250adb3b41&hint=978a7273-8cff-45bb-b033-c4040a8a39dd&sourcetime=1778059741251
```

Detailed environment notes are in `docs/19-development-environment.md`.
