# DocCentral / e-Pisarnica — GitHub Documentation Repository

## 1. Purpose

This repository contains documentation, requirements, architecture notes, templates, checklists and Claude Code skills for the DocCentral / e-Pisarnica solution.

The goal is to provide a GitHub-ready knowledge base that can be used as the foundation for developing a new, improved enterprise version of the application.

---

## 2. Project summary

DocCentral is an electronic registry and document archive solution based on Microsoft Power Platform and SharePoint Online.

Core platform:

- Power Apps Canvas application
- Power Automate flows
- SharePoint Online lists and document libraries
- Microsoft 365 / Entra ID groups
- App Config list for dictionaries and configuration

Main business capabilities:

- document entry
- document metadata management
- document registration
- unique registry number / `DelovodniBroj`
- document archiving
- partner data usage
- organizational-unit based access
- SharePoint item/file permission control

---

## 3. Critical project principles

These rules are mandatory for the new version:

1. Users have Read Only permissions in SharePoint.
2. Create/Edit/Delete operations go through Power Automate.
3. Power Automate writes under service account.
4. `DelovodniBroj` must be unique and concurrency-safe.
5. Canvas app must be responsive.
6. Serbian is the default language.
7. English must be supported.
8. App Config should be used for dictionaries and configuration.
9. SharePoint backend permissions must enforce access.
10. UI filtering in Canvas app is not enough for security.
11. Critical operations must be logged.
12. Failed registry number attempts must be logged.

---

## 4. Repository structure

```text
.
├── README.md
├── requirements/
│   ├── functional-requirements.md
│   └── non-functional-requirements.md
├── architecture/
│   ├── target-architecture.md
│   └── concurrency-design.md
├── power-platform/
│   ├── power-apps-rules.md
│   └── power-automate-rules.md
├── security/
│   └── security-model.md
├── testing/
│   ├── test-plan.md
│   └── acceptance-criteria.md
├── migration/
│   └── migration-plan.md
├── operations/
│   ├── deployment-guide.md
│   └── runbook.md
├── templates/
│   ├── flow-response-template.md
│   ├── error-log-template.md
│   └── sharepoint-list-definition-template.md
├── checklists/
│   ├── pre-development-checklist.md
│   └── code-review-checklist.md
├── claude-code/
│   └── README.md
├── skills/
│   ├── doccentral-architecture-skill.md
│   ├── power-apps-skill.md
│   ├── power-automate-skill.md
│   ├── concurrency-skill.md
│   └── security-skill.md
└── data-model/
    └── existing SharePoint list documentation files
```

---

## 5. Known data model areas

Known or expected SharePoint assets:

- `Svi predmeti`
- `Partneri`
- `App Config`
- `Rezervisani Brojevi`
- document libraries
- email/shared document related libraries or lists where applicable

Important clarification:

- XML list exports are used for understanding SharePoint schemas.
- JSON in App Config contains dictionaries and configuration read by the application.
- Internal column names are often the same as display names without spaces, but must be verified from exports.
- Required fields are not fully defined yet.

---

## 6. DelovodniBroj / registry number

This is the most critical enterprise requirement.

Known facts:

- App Config contains `Delovodne knjige`.
- Next registry number is stored and incremented there.
- `Rezervisani Brojevi` list exists.
- `DelovodniBroj` is unique.
- ETag is not currently used.
- Retry logic should be implemented.
- Audit of failed attempts should be implemented.
- Every year the registry book is locked.
- Archive book is created for registered documents in archived status.

Target rule:

- Number generation/reservation must be handled by backend logic.
- Two users must never receive the same `DelovodniBroj`.

Detailed design:

- `architecture/concurrency-design.md`

---

## 7. Security model

Known rules:

- Standard users have Read Only permissions in SharePoint.
- Power Automate writes under service account.
- Document access depends on organizational units.
- Entra group mapping depends on each client.
- Item/folder/file permissions use break inheritance.
- Service account has read/write rights.
- Owners have read/write rights.
- Members/Viewers have Read rights.
- There is no separate admin group currently.
- Access is enforced both in Canvas app and SharePoint.
- Backend permissions hide files/items, not only UI filtering.

Detailed model:

- `security/security-model.md`

---

## 8. Power Apps direction

Claude Code may decide:

- exact screen count
- screen names
- controls
- responsive layout
- UI components

But it must respect:

- no direct SharePoint writes for Create/Edit/Delete
- App Config usage
- Serbian/English support
- responsive layout
- standardized flow response handling
- delegable or server-side filtering
- security rules

Detailed rules:

- `power-platform/power-apps-rules.md`

---

## 9. Power Automate direction

Power Automate is the backend layer.

It should handle:

- create/update/delete operations
- registration
- registry number reservation
- archiving
- permission assignment
- audit/error logging

Detailed rules:

- `power-platform/power-automate-rules.md`

---

## 10. Out of scope for now

The following areas are intentionally not fully specified yet:

- SAP import implementation details
- external import format
- client-specific Entra group map
- final approval process details
- PDF generation details
- final production deployment values

These should be added later when confirmed.

---

## 11. How Claude Code should use this repository

Claude Code should:

1. Read this README.
2. Read requirements.
3. Read architecture and concurrency design.
4. Read Power Apps and Power Automate rules.
5. Read security model.
6. Read testing and acceptance criteria.
7. Use skills from `skills/` folder as project-specific behavior instructions.
8. Mark unknowns as `NEPOZNATO` / `UNKNOWN`.
9. Avoid inventing undocumented facts.
10. Generate implementation that satisfies acceptance criteria.

---

## 12. Current readiness level

This repository is ready for:

- documentation consolidation
- architecture planning
- Claude Code briefing
- initial development planning
- test planning

Before full build, verify:

- latest Power Platform solution export
- complete flow inventory
- complete Canvas app formulas
- complete SharePoint schemas
- App Config active keys
- client-specific group mapping
- deployment environment details
