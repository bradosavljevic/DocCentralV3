# Non-Functional Requirements — DocCentral / e-Pisarnica

## 1. Purpose

This document defines non-functional requirements for the next DocCentral version.

These requirements are mandatory because the solution is expected to behave as an enterprise-grade registry and archive system, not only as a basic Power Apps form.

---

## 2. Reliability

### NFR-001 — Unique registry number

The system must never create duplicate `DelovodniBroj` values.

This applies even when:

- multiple users register documents at the same time
- a Power Automate flow is retried
- SharePoint responses are delayed
- one registration attempt fails halfway

### NFR-002 — Controlled retries

Critical write operations must support controlled retry logic.

Examples:

- reserving registry number
- updating App Config next number
- writing registered document metadata
- applying item/file permissions

Retries must not create duplicate records or duplicate numbers.

### NFR-003 — Failure traceability

Every critical failure must be traceable through logs.

At minimum, logs should identify:

- user
- operation
- entity/item
- flow name
- run id or correlation id
- error code
- error message
- timestamp

---

## 3. Security

### NFR-010 — SharePoint Read Only for users

Standard users must have Read Only permissions on SharePoint lists and libraries.

Create/Edit/Delete operations must be executed through Power Automate under a controlled service account.

### NFR-011 — Backend permission enforcement

Security must not depend only on Canvas app UI filtering.

SharePoint item, folder and file permissions must enforce real access control.

### NFR-012 — Service account control

The service account must be documented and used consistently for backend operations.

The service account should have required write permissions and must not be casually replaced by personal user accounts.

### NFR-013 — Client-specific group mapping

Entra group mapping depends on each client.

The solution must support configurable organizational-unit based access instead of hardcoding one tenant-specific group structure.

---

## 4. Performance

### NFR-020 — Large list readiness

The solution must be designed for SharePoint lists that can grow beyond small test volumes.

Power Apps formulas and Power Automate queries must consider:

- SharePoint delegation
- indexed columns
- server-side filtering
- pagination
- payload size
- avoiding unnecessary full list loads

### NFR-021 — Responsive Canvas app

The Canvas app must be fully responsive and usable on supported screen sizes.

### NFR-022 — Fast initial load

The app should not load all large SharePoint lists on startup unless absolutely required.

Recommended pattern:

- load configuration first
- load user context and permissions
- load only required dashboard/list data
- use lazy loading for detail screens

---

## 5. Maintainability

### NFR-030 — GitHub-ready documentation

All important technical decisions must be documented in markdown files.

Documentation should include:

- architecture
- data model
- security model
- Power Apps rules
- Power Automate rules
- deployment guide
- runbook
- test plan

### NFR-031 — Naming consistency

All new components should follow consistent naming rules.

This includes:

- Canvas app screens
- controls
- variables
- collections
- flows
- SharePoint columns
- environment variables
- connection references

### NFR-032 — No hidden critical logic

Critical business logic must not be hidden only in scattered Canvas formulas.

Where possible, critical operations should be centralized in Power Automate or documented named formulas/components.

---

## 6. Observability

### NFR-040 — Audit log

The enterprise version should include audit logging for:

- document creation
- document update
- document registration
- registry number reservation
- archive action
- permission changes
- failed registration attempts
- partner changes
- critical flow errors

### NFR-041 — Error log

The enterprise version should include centralized error logging.

A recommended model is defined in:

- `templates/error-log-template.md`

---

## 7. Localization

### NFR-050 — Serbian and English

The new solution must support Serbian and English.

Serbian is the default language.

### NFR-051 — No hardcoded labels where avoidable

User-facing text should be stored in a translation dictionary or configuration structure when practical.

---

## 8. Power Platform constraints

### NFR-060 — Standard connectors preferred

Use standard connectors unless the project explicitly approves premium connectors.

### NFR-061 — Power Automate timeout awareness

Power Apps-triggered flows must return in a reasonable time.

Long operations should use an asynchronous pattern.

### NFR-062 — Environment portability

The solution should use documented environment variables and connection references so it can be imported into another environment with minimal manual changes.
