# Migration Plan — DocCentral / e-Pisarnica

## 1. Purpose

This document defines the migration approach from the existing DocCentral implementation to the new version.

The migration plan must preserve existing business data, documents, registry numbers and archive integrity.

---

## 2. Migration principles

- Do not change existing `DelovodniBroj` values.
- Do not lose document metadata.
- Do not lose document files.
- Preserve archive integrity.
- Validate permissions after migration.
- Keep rollback option where practical.
- Test migration in non-production environment first.

---

## 3. Data to migrate or preserve

Known data areas:

- `Svi predmeti`
- `Partneri`
- `App Config`
- `Rezervisani Brojevi`
- document libraries
- existing permissions
- existing archive records

---

## 4. Pre-migration tasks

Checklist:

- Export existing solution.
- Export SharePoint list schemas.
- Export App Config.
- Export or document document libraries.
- Document service account.
- Document connection references.
- Document environment variables.
- Confirm current production URL/site.
- Confirm target environment/site.
- Backup critical SharePoint lists and libraries.

---

## 5. Migration approach

### Option A — In-place upgrade

Use same SharePoint lists/libraries and deploy new app/flows.

Pros:

- less data movement
- keeps existing URLs where possible

Cons:

- higher risk if production data model is changed directly
- rollback can be harder

### Option B — New site/new lists with migration

Create new SharePoint structure and migrate data.

Pros:

- cleaner target model
- easier parallel testing

Cons:

- more migration work
- URLs and permissions must be recreated

Recommended decision depends on final data model and client constraints.

---

## 6. Registry number migration

Rules:

- Existing `DelovodniBroj` values must remain unchanged.
- App Config next number must be validated against existing max value.
- Closed years must remain locked.
- Archive book records must remain consistent.
- `Rezervisani Brojevi` must be reviewed for stale reservations.

---

## 7. App Config migration

Steps:

1. Export existing App Config.
2. Identify active keys and dictionaries.
3. Remove obsolete values only after confirmation.
4. Add new required configuration keys.
5. Validate app loads configuration correctly.

---

## 8. Permission migration

Steps:

1. Identify existing item/file permissions.
2. Validate organizational-unit mapping.
3. Confirm service account permissions.
4. Apply new permission rules where needed.
5. Test unauthorized direct access.

---

## 9. Rollback plan

Rollback should include:

- restore previous app version
- restore previous flow versions
- restore SharePoint backup if schema/data changed
- disable new flows if they conflict with old implementation
- document any records created during failed migration window

---

## 10. Production cutover checklist

- Solution imported successfully.
- Connection references configured.
- Environment variables configured.
- Service account tested.
- App Config loaded.
- Test document created.
- Test document registered.
- Concurrent registration test passed.
- Permissions test passed.
- Audit/error logging verified.
- Users informed.
