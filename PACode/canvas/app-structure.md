# Canvas App structure: DocCentralV3

## App identity

| Property | Value |
|---|---|
| Display name | DocCentralV3 |
| Logical name | gpdoccen_doccentralv3_d98ba |
| Solution | DocCentralV3 |

## Architectural constraints

- Canvas App is UI only. No direct SharePoint writes from the app.
- All user-initiated create/update/delete operations go through Power Automate flows.
- Users have SharePoint Read Only access. Item-level visibility is enforced on the backend.
- App Config is loaded once on app start and cached in a collection.
- No live search on every character. Search triggers on button press or after debounce.
- No cross-screen control references. Use global variables and collections to share state.
- No nested galleries unless explicitly justified and documented.
- All Power Apps formulas use `IfError` around Patch and flow calls.
- Naming conventions: `gbl` (global variable), `loc` (context variable), `col` (collection), `scr` (screen), `gal` (gallery), `frm` (form), `btn` (button), `txt` (text input), `cmb` (combobox), `dp` (date picker), `cmp` (component).

## Screen inventory

| Screen name | Display label | Purpose |
|---|---|---|
| scrNoviPredmet | Novi predmet | Document registration form |
| scrZaOdobrenje | Za odobrenje | Documents waiting for current user's approval |
| scrPodsetnici | Podsetnici | Reminders — view, create, edit, delete |
| scrArhiviranje | Arhiviranje | Archive documents (Zavedeno → Arhivirano) |
| scrZakljucenjeGodine | Zaključenje godine | Year closing with precondition checks |
| scrPartneri | Partneri | Partner list, create, edit, soft-delete |
| scrAdministracija | Administracija | Codebook/App Config management |
| scrLoading | (hidden) | App initialization, App Config loading |

Note: There is no dashboard screen. There is no general document list screen. Documents are reviewed directly in SharePoint.

## Navigation model

Navigation is driven by a vertical side navigation component or top navigation bar (final UX design to be determined during screen generation).

Navigation items:
1. Novi predmet
2. Za odobrenje
3. Podsetnici
4. Arhiviranje
5. Zaključenje godine
6. Partneri
7. Administracija

Each navigation item maps to one screen.
Active screen is tracked in `gblCurrentScreen` global variable.
Navigation triggers `Navigate()` with a `ScreenTransition.None` or `Fade` transition.

## Global variables

| Name | Type | Set in | Purpose |
|---|---|---|---|
| gblCurrentUserEmail | Text | App.OnStart | Current user email from `User().Email` |
| gblCurrentUserDisplayName | Text | App.OnStart | Current user display name from `User().FullName` |
| gblCurrentScreen | Text | Navigation actions | Tracks active screen name |
| gblAppConfigLoaded | Boolean | App.OnStart | True when App Config collection is fully loaded |
| gblActiveYear | Number | App.OnStart | Active registry year from App Config |
| gblIsAdmin | Boolean | App.OnStart | True if current user is in admin group (from App Config group mapping) |
| gblErrorMessage | Text | Error handlers | Last error message for display |
| gblIsLoading | Boolean | Flow calls | True while a flow is executing |

## Collections

| Name | Loaded in | Source | Purpose |
|---|---|---|---|
| colAppConfig | App.OnStart | App Config SharePoint list | All configuration and codebook entries |
| colDocumentTypes | App.OnStart | Filtered from colAppConfig | Document type lookup |
| colStatuses | App.OnStart | Filtered from colAppConfig | Document status lookup |
| colArchiveSigns | App.OnStart | Filtered from colAppConfig | Archive signs (Arhivski znaci) |
| colProcessConfig | App.OnStart | Filtered from colAppConfig | Process configuration per document type |
| colUserGroups | App.OnStart | Office365Groups connector | Current user's Entra group IDs |
| colPartneri | scrPartneri.OnVisible | Partneri list | Active (non-deleted) partners |
| colRezervisaniBrojevi | scrNoviPredmet.OnVisible | Rezervisani brojevi list | Available reserved numbers |
| colZaOdobrenje | scrZaOdobrenje.OnVisible | ApprovalSteps list | Approval steps pending for current user |
| colPodsetnici | scrPodsetnici.OnVisible | Podsetnici list | User's reminders |
| colArhiviranje | scrArhiviranje.OnVisible | Svi predmeti list | Documents in Zavedeno status for current year |

## App.OnStart sequence

```
1. Set gblCurrentUserEmail = User().Email
2. Set gblCurrentUserDisplayName = User().FullName
3. Set gblAppConfigLoaded = false
4. Set gblIsLoading = true
5. ClearCollect colAppConfig from App Config SharePoint list (all items, delegable)
6. Set gblActiveYear from colAppConfig (filter by UNKNOWN active year key)
7. Set colDocumentTypes = Filter(colAppConfig, Category = UNKNOWN_DOCTYPE_CATEGORY_KEY)
8. Set colStatuses = Filter(colAppConfig, Category = UNKNOWN_STATUS_CATEGORY_KEY)
9. Set colArchiveSigns = Filter(colAppConfig, Category = UNKNOWN_ARCHIVESIGN_CATEGORY_KEY)
10. Set colProcessConfig = Filter(colAppConfig, Category = UNKNOWN_PROCESSCONFIG_CATEGORY_KEY)
11. ClearCollect colUserGroups from Office365Groups.ListMyMemberOf() — extract group IDs
12. Set gblIsAdmin based on whether gblCurrentUserEmail or colUserGroups match admin group from App Config (UNKNOWN key)
13. Set gblAppConfigLoaded = true
14. Set gblIsLoading = false
15. Navigate(scrNoviPredmet)
```

App Config category keys are UNKNOWN until AppConfig.csv is confirmed. All filters use dynamic keys read from App Config, not hardcoded strings.

## App Config loading pattern

App Config items are structured as key-value entries with a category/type discriminator.
The app reads all App Config items into `colAppConfig` once.
All screens that need configuration values look up from `colAppConfig` or from pre-filtered sub-collections.

Example lookup pattern (exact column names UNKNOWN):
```
LookUp(colAppConfig, ConfigKey = "UNKNOWN_KEY", ConfigValue)
```

No screen or formula may hardcode a configuration value that belongs in App Config.

## Error handling pattern

All flow calls in the Canvas App must follow this pattern:

```
Set(gblIsLoading, true);
IfError(
    Set(varFlowResult, CF_FlowName.Run(params)),
    Set(gblErrorMessage, "Greška: " & FirstError.Message);
    Notify(gblErrorMessage, NotificationType.Error)
);
If(
    !IsError(varFlowResult) && varFlowResult.success,
    // success path
    Notify("Operacija je uspešno izvršena.", NotificationType.Success),
    // failure path
    Notify(varFlowResult.message, NotificationType.Error)
);
Set(gblIsLoading, false)
```

## Screen: scrNoviPredmet (Novi predmet)

### Purpose
Document registration form. Allows user to enter document metadata, select a partner, upload the main file, optionally upload attachments, and choose between generating a new registry number or using a reserved one.

### Key controls

| Control | Type | Purpose |
|---|---|---|
| cmbDocumentType | Combobox | Select document type from colDocumentTypes |
| txtSubject | Text input | Document subject/title |
| txtDescription | Text input (multi-line) | Description |
| cmbPartner | Combobox | Select partner from colPartneri |
| dpFilingDate | Date picker | Filing date (only visible if useReservedNumber = true) |
| togUseReservedNumber | Toggle | Switch between new number / reserved number |
| cmbRezervisaniBroj | Combobox | Select from colRezervisaniBrojevi (visible only when toggle = true) |
| attMainFile | Attachment control | Upload main document file |
| attAttachments | Attachment control | Upload optional attachments |
| btnZavedi | Button | Submit the form |
| lblDelovodniBrojPreview | Label | Preview area (shows reserved number if selected) |

### Validation before submit

All validation in `btnZavedi.OnSelect` before calling flow:
- `cmbDocumentType` is not blank.
- `txtSubject` is not blank.
- `attMainFile` has at least one file.
- If `togUseReservedNumber = true`: `cmbRezervisaniBroj` is not blank.
- If `togUseReservedNumber = true`: `dpFilingDate` is not blank.

Show `Notify()` for each failed validation. Do not call flow until all validations pass.

### Flow call

On successful validation:
```
IfError(
    Set(varCreateResult, CF_DocCentralV3_CreateDocument.Run(
        cmbDocumentType.Selected.Value,
        {
            partnerNaziv: cmbPartner.Selected.Naziv,
            partnerId: cmbPartner.Selected.ID,
            partnerPIBSnapshot: cmbPartner.Selected.PIB,
            ...
        },
        togUseReservedNumber.Value,
        If(togUseReservedNumber.Value, cmbRezervisaniBroj.Selected.ID, Blank()),
        If(togUseReservedNumber.Value, Text(dpFilingDate.SelectedDate, "yyyy-mm-dd"), ""),
        ...
    )),
    Notify("Greška pri pozivu servisa.", NotificationType.Error)
);
```

### Post-submit behavior

On success: Reset all form controls. Show success notification with `DelovodniBroj`. Reload `colRezervisaniBrojevi`.
On failure: Show error notification with message from flow response. Do not reset form (allow user to correct and retry).

## Screen: scrZaOdobrenje (Za odobrenje)

### Purpose
Shows all approval steps assigned to the current user (by email or group membership) that are still Pending.

### Data source
`colZaOdobrenje` — filtered from `ApprovalSteps` list:
- `StepStatus = "Pending"` AND (`AssigneeUserEmail = gblCurrentUserEmail` OR `AssigneeGroupId` is in `colUserGroups`)

### Key controls

| Control | Type | Purpose |
|---|---|---|
| galZaOdobrenje | Gallery | List of pending approval steps |
| lblDelovodniBroj | Label | Registry number |
| lblDocumentType | Label | Document type |
| lblInitiator | Label | Who submitted the document |
| lblStepNumber | Label | Approval step sequence |
| btnOdobri | Button | Approve the document |
| btnOdbij | Button | Reject the document |
| txtKomentar | Text input | Optional comment for approval/rejection decision |

### Flow calls
- `btnOdobri.OnSelect` → call `CF_DocCentralV3_ProcessApprovalResponse` with outcome = `Approved`
- `btnOdbij.OnSelect` → call `CF_DocCentralV3_ProcessApprovalResponse` with outcome = `Rejected`

After response: reload `colZaOdobrenje`.

## Screen: scrPodsetnici (Podsetnici)

### Purpose
Create, view, edit, and delete reminders. Reminders are stored in `Podsetnici` list.

### Rules
- Reminders are read from SharePoint (Read Only user access).
- Create/edit/delete go through flows (UNKNOWN — no flow design yet for reminder CRUD; may use Patch if service account writes are not required for reminder data, or require a flow if the list is restricted to service account writes).
- Reminder date must be in the future (client-side validation).
- Recipient list can include multiple users (comma-separated emails or a People field — UNKNOWN from list schema).

### Open item
Reminder CRUD flow not yet designed. If the `Podsetnici` list requires service account write access, a flow must be added to the inventory. Otherwise, direct Patch from Canvas App to the list is acceptable IF the list grants user write access.

## Screen: scrArhiviranje (Arhiviranje)

### Purpose
Shows all documents in `Zavedeno` status for the active year. Allows archiving one or more documents with archive sign selection.

### Data source
Filter `Svi predmeti` where `Stanje = "Zavedeno"` AND `FilingYear = gblActiveYear`.

This filter must be delegable. Use SharePoint filtering via `Filter()` with supported OData operators.

### Key controls

| Control | Type | Purpose |
|---|---|---|
| galArhiviranje | Gallery | List of documents eligible for archiving |
| cmbArhivskiZnak | Combobox | Select archive sign from colArchiveSigns |
| chkSelectAll | Checkbox | Select/deselect all visible documents |
| btnArhiviraj | Button | Archive selected documents |

### Flow call
`btnArhiviraj.OnSelect` → call `CF_DocCentralV3_ArchiveDocument` for each selected item (or pass a list of IDs — to be designed in Phase 1 flow design).

## Screen: scrZakljucenjeGodine (Zaključenje godine)

### Purpose
Allows administrator to close the active registry year. Displays precondition checks before allowing the action.

### Precondition checks (displayed to user before confirmation)

| Check | How evaluated |
|---|---|
| All documents Arhivirano | Count of Svi predmeti where Stanje ≠ Arhivirano for active year = 0 |
| No reserved numbers | Count of RezervisaniBrojevi for active year = 0 |

Both checks must pass before the close button is enabled.

### Key controls

| Control | Type | Purpose |
|---|---|---|
| lblCheckDocuments | Label | Shows count of non-archived documents |
| lblCheckReservedNumbers | Label | Shows count of remaining reserved numbers |
| icnCheckDocuments | Icon | Green check / red X based on count |
| icnCheckReservedNumbers | Icon | Green check / red X based on count |
| btnZakljuciGodinu | Button | Disabled unless all checks pass |

### Flow call
`btnZakljuciGodinu.OnSelect` → call `CF_DocCentralV3_CloseRegistryYear`.

After success: refresh `gblActiveYear` from App Config. Show confirmation. Navigate away.

### Important
No undo. No re-open option. UI must clearly communicate the irreversibility.

## Screen: scrPartneri (Partneri)

### Purpose
Full partner management: view, search, create, edit, soft-delete.

### Rules
- Search triggers on button press, not on every character.
- Deleted partners show `IsDeleted = true` in the list. They are excluded from the gallery via Filter.
- Creating/editing a partner goes through a flow or Patch if user write access is granted on the Partneri list.
- Soft-delete: flow updates `IsDeleted = true` (or equivalent) rather than physically deleting the item.

### Key controls

| Control | Type | Purpose |
|---|---|---|
| txtSearchPartner | Text input | Search text |
| btnSearch | Button | Trigger search |
| galPartneri | Gallery | Filtered partner list |
| frmPartner | Form (Edit/New) | Create/edit partner |
| btnNoviPartner | Button | Open form in New mode |
| btnSacuvaj | Button | Save partner |
| btnObrisi | Button | Soft-delete partner |

## Screen: scrAdministracija (Administracija)

### Purpose
Manage App Config codebooks — view and edit configuration entries. Export App Config.

### Rules
- Admin-only screen. Visible and accessible only if `gblIsAdmin = true`.
- Non-admin users should not be able to navigate to this screen.
- Editing codebook values goes through a flow (write via service account).
- Export App Config triggers `CF_DocCentralV3_ExportAppConfig`.

### Key controls

| Control | Type | Purpose |
|---|---|---|
| galAppConfig | Gallery | List of App Config entries |
| frmAppConfigEdit | Form | Edit a single config entry |
| btnExport | Button | Trigger ExportAppConfig flow |

## Responsive design

- App uses responsive layout (not fixed pixel sizes).
- Use `App.Width`, `App.Height`, `Parent.Width` for control sizing.
- Navigation collapses to icons-only on narrow screens.
- Minimum target width: 1024px (web/tablet). Mobile is secondary.

## App Checker requirements

- No inaccessible controls.
- No deprecated functions.
- All delegable filters use SharePoint-supported OData operators.
- No `CountRows` on non-delegable filtered results above 500 items without explicit `ThisItem` context.
- All galleries have explicit `Items` set to a named collection or a directly delegable Filter expression.

## Open items

| Item | Status |
|---|---|
| App Config category/key column names | UNKNOWN — required before App.OnStart formula can be finalized |
| Svi predmeti FilingYear column name | UNKNOWN |
| Partneri IsDeleted column name | UNKNOWN |
| Podsetnici CRUD access model (flow vs direct Patch) | TO BE DECIDED |
| ApprovalSteps list environment variable logical name | UNKNOWN |
| Admin group App Config key | UNKNOWN |
| Navigation component type (side bar vs top bar) | TO BE DECIDED during screen generation |
