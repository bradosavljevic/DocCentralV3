# Functional Requirements — DocCentral / e-Pisarnica

## 1. Purpose

This document defines the functional requirements for the next version of DocCentral / e-Pisarnica.

DocCentral is an electronic registry and document archive solution based on:

- Power Apps Canvas application
- Power Automate backend layer
- SharePoint Online lists and document libraries
- Microsoft 365 / Entra ID groups
- App Config list for dictionaries and configuration

The solution must support controlled document registration, document metadata management, partner data, organizational-unit based access, archiving and auditability.

---

## 2. Core business concepts

### 2.1 Document

A document represents a business record that can be entered, reviewed, registered and archived.

Expected document capabilities:

- Create new document metadata
- Attach or associate files
- Edit document metadata before final registration, depending on status and permissions
- Register the document and assign a unique `DelovodniBroj`
- Archive the document
- View document history and current status
- Control visibility based on organizational unit and SharePoint permissions

### 2.2 Registry number / Delovodni broj

`DelovodniBroj` is the official registry number.

Mandatory rules:

- It must be unique.
- It must never be duplicated, even if multiple users register documents at the same time.
- It must be generated or reserved through server-side logic, not by trusting only Canvas app logic.
- The final generated number must be persisted in SharePoint.
- Failed attempts must be logged.
- Retry logic must exist for concurrency conflicts.

Current known facts:

- App Config contains configuration for `Delovodne knjige`.
- The next number is stored in configuration and incremented.
- A `Rezervisani Brojevi` list exists.
- `DelovodniBroj` is unique.
- Current implementation does not use ETag.
- Current implementation does not have complete retry/audit logic for failed number allocation.

### 2.3 Partner

Partner records are used for document metadata and invoice/document processing.

Partner capabilities:

- Read partner data from SharePoint list `Partneri`.
- Search and select partner in document forms.
- Identify partner by PIB where applicable.
- Support synchronization/update from external source when available.

SAP/external import details are intentionally out of scope for now.

### 2.4 App Config

App Config contains dictionaries and configuration used by the application.

The app should use App Config for:

- Document types
- Registry books
- Statuses
- Dropdown values
- Application-level configuration
- Potential multilingual resources
- Potential UI behavior configuration

Hardcoding dictionaries in Canvas formulas should be avoided where App Config already provides the value.

---

## 3. Document lifecycle requirements

### 3.1 Create document

The user must be able to create a new document through the Canvas app.

Required behavior:

- User opens the create document experience.
- App loads required dictionaries from App Config.
- App loads relevant SharePoint reference data.
- User enters metadata.
- User selects or adds required partner information where applicable.
- User attaches or links document files where required.
- App validates required fields before submission.
- App calls Power Automate for Create operation.
- Power Automate writes to SharePoint under service account.
- Flow returns a standardized response to Canvas app.
- App displays success or error message.

### 3.2 Edit document

The user must be able to edit documents where business rules allow it.

Required behavior:

- App opens selected document in edit mode.
- Only fields allowed for the current status and role are editable.
- App validates changes before submission.
- App calls Power Automate for Update operation.
- Power Automate writes to SharePoint under service account.
- Changes are logged where audit logging exists.

### 3.3 View document

The user must be able to view documents they have permission to access.

Required behavior:

- User sees only documents permitted by SharePoint permissions and business filtering.
- UI filtering must not be the only security mechanism.
- Backend permissions must prevent unauthorized access.
- Document files and list items must both be secured.

### 3.4 Register document

Registration is the process of assigning final `DelovodniBroj` and marking the document as officially registered.

Required behavior:

- User clicks register / `Zavedi`.
- Canvas app calls a dedicated Power Automate flow.
- Flow performs server-side validation.
- Flow reserves or generates a unique `DelovodniBroj`.
- Flow writes the number to the document item.
- Flow updates related status fields.
- Flow applies or updates item/file permissions if needed.
- Flow logs success or failure.
- Flow returns standardized response.

### 3.5 Archive document

Archiving marks the document as final and moves/keeps it in archive state.

Required behavior:

- Only authorized users can archive.
- Archived documents must preserve metadata and final document files.
- Archived documents must be included in archive book/year closure logic.
- Archived documents must remain searchable according to permissions.

### 3.6 Year closure / archive book

Known requirement:

- Each year the registry book is locked.
- An archive book is created for all registered documents in archived status.

Required behavior:

- The system must prevent accidental changes to closed registry periods.
- Archive book creation must be traceable.
- Existing `DelovodniBroj` values must remain unchanged.

---

## 4. SharePoint data requirements

The solution uses SharePoint as the data layer.

Known lists/libraries include:

- `Svi predmeti`
- `Partneri`
- `App Config`
- `Rezervisani Brojevi`
- document libraries for stored files

Rules:

- Users have Read Only permissions in SharePoint.
- Writes must go through Power Automate under service account.
- Item/folder/file permissions must be applied based on organizational unit/group rules.
- Internal names are often the same as display names without spaces, but this must be verified from schema exports.
- Required fields are currently not fully defined and should be documented explicitly when implemented.

---

## 5. Security-related functional requirements

The solution must support organizational-unit based access.

Known rules:

- Access depends on organizational units.
- Entra group mapping depends on each client.
- Service account has read/write rights.
- Owners have read/write rights.
- Members/Viewers have read rights.
- There is no separate admin group currently.
- Access is enforced in both Canvas app and SharePoint.
- Backend permissions must hide files/items, not only UI filtering.

---

## 6. Multilingual requirements

The new Canvas app must support:

- Serbian as default language
- English as additional language

Expected behavior:

- Labels, buttons, messages and validation texts must support translation.
- Translation should be centralized, preferably through configuration or translation dictionary.
- Hardcoded user-facing text should be minimized.

---

## 7. Power Apps behavior requirements

The Canvas app must:

- Be responsive.
- Use consistent layout patterns.
- Avoid direct SharePoint writes for Create/Edit/Delete.
- Use Power Automate for write operations.
- Use App Config for dictionaries and configuration.
- Handle loading, success and error states consistently.
- Use delegable queries where possible.
- Avoid non-delegable patterns over large lists.

Claude Code can decide:

- number of screens
- screen names
- controls per screen
- UX layout

But it must respect the functional requirements and existing solution analysis.

---

## 8. Out of scope for now

The following should be documented but not implemented unless explicitly requested:

- SAP import implementation details
- external API integration details
- client-specific Entra group mapping
- final approval matrix, unless later defined
- PDF generation implementation details, unless later defined
