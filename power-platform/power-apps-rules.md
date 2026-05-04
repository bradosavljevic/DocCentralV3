# Power Apps Canvas Rules — DocCentral

## 1. Purpose

This document defines rules for building the new DocCentral Canvas app.

Claude Code may decide the final screen count, screen names and control structure, but must respect these rules.

---

## 2. Write operations

### Mandatory rule

Canvas app must not directly write to SharePoint for:

- Create
- Edit
- Delete
- Register / `Zavedi`
- Archive
- Permission changes

These operations must go through Power Automate.

Reason:

- users have Read Only SharePoint rights
- Power Automate runs under service account
- validation and audit are centralized
- concurrency-safe `DelovodniBroj` logic must be server-side

---

## 3. Read operations

Canvas app may read data through:

- SharePoint connector
- Power Automate returning JSON
- Office 365 Users connector
- Microsoft 365 Groups connector

Use direct SharePoint read when:

- the query is delegable
- the result set is controlled
- the user has permission to read the data

Use Power Automate read when:

- data must be transformed
- multiple sources must be joined
- security context must be evaluated server-side
- SharePoint delegation would be problematic

---

## 4. Responsiveness

The app must be responsive.

Recommended layout:

- use containers
- avoid fixed-position layouts where possible
- support desktop-first design but do not break on smaller screens
- use reusable components for header, navigation, forms, messages and loading state

---

## 5. Localization

Serbian is default language.

English must be supported.

Rules:

- avoid hardcoded labels where practical
- use translation dictionary/configuration
- centralize user-facing messages
- validation messages must also support localization

---

## 6. App Config usage

App Config should be loaded and used for:

- dictionaries
- document types
- registry books
- statuses
- configuration flags
- potential visibility rules
- potential labels/translations

Canvas app should not duplicate App Config values as hardcoded collections unless required for performance and documented.

---

## 7. Variables and collections

Recommended naming:

| Type | Prefix | Example |
|---|---|---|
| Global variable | `gbl` | `gblCurrentUser` |
| Local/context variable | `loc` | `locShowSpinner` |
| Collection | `col` | `colDocuments` |
| Component property | `cmp` or descriptive | `cmpHeader` |
| Named formula | `nf` | `nfTranslate` |

Existing app may use other patterns. New code should be consistent and documented.

---

## 8. Error handling

Every flow call must handle:

- success response
- business validation error
- technical error
- timeout or missing response

Flow response should follow:

- `templates/flow-response-template.md`

---

## 9. Loading state

Every backend call should show clear loading state.

Rules:

- prevent duplicate clicks during critical operations
- disable register/save buttons while operation is running
- show success or failure after response

---

## 10. Delegation

Avoid non-delegable filters over large SharePoint lists.

Preferred practices:

- filter by indexed columns
- use exact matches where possible
- avoid loading all documents into local collection
- use server-side filtering through Power Automate where necessary

---

## 11. Security in UI

The app should hide or disable actions based on user permissions.

However, UI rules are not sufficient.

Backend flow and SharePoint permissions must also enforce access.

---

## 12. Forms

Visible fields should depend on:

- document type
- status
- user role/group
- App Config where applicable

Only fields visible/allowed in the current mode should be editable.

Modes:

- New
- Edit
- View
- Archive/Read-only

---

## 13. Registry number behavior

Canvas app may display a preview if useful, but final `DelovodniBroj` must come from backend confirmation.

The app must not assume that the locally calculated next number is final.
