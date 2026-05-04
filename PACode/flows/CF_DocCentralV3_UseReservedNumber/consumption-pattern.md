# Reserved number consumption pattern: CF_DocCentralV3_UseReservedNumber

## Overview

A reserved number is a pre-allocated registry number that a user has claimed for a future document.
The number is stored in the RezervisaniBrojevi list until the document is actually registered.

This flow handles **validation only**. The consumption (deletion) is deliberately deferred to the
parent flow (CF_DocCentralV3_CreateDocument) and occurs only after the document is fully created.

---

## Why validation and deletion are separated

If this flow deleted the reserved number and then document creation failed, the number
would be permanently lost — the document does not exist but the number is gone.

The two-phase approach:
1. **Phase 1 — This flow:** Validate the reserved number (exists, correct year, not duplicate). Return the delovodniBroj. Do not delete.
2. **Phase 2 — CF_DocCentralV3_CreateDocument:** Create the Svi predmeti item and upload the file. If both succeed, then delete the reserved number from RezervisaniBrojevi.

If Phase 2 fails at any point, the reserved number remains in RezervisaniBrojevi and the
user can retry registration.

---

## Full call sequence from CF_DocCentralV3_CreateDocument

```
CF_DocCentralV3_CreateDocument
│
├── IF reservedNumberId > 0:
│   │
│   ├── CALL CF_DocCentralV3_UseReservedNumber
│   │     input: { reservedNumberId, requestedYear, initiatorEmail, correlationId }
│   │
│   ├── IF success = false → STOP, return failure to Canvas App
│   │
│   └── IF success = true → capture delovodniBroj from response
│
├── IF reservedNumberId = 0:
│   └── CALL CF_DocCentralV3_GenerateRegistryNumber
│         (auto-generate a new number)
│
├── Create item in Svi predmeti (DelovodniBroj = validated/generated number)
│
├── Upload document file to document library
│
├── IF any creation step fails → STOP, return failure
│   (reserved number NOT deleted — remains available)
│
├── IF all creation steps succeed:
│   │
│   ├── IF reservedNumberId > 0:
│   │   ├── DELETE RezervisaniBrojevi item (ID = reservedNumberId)
│   │   └── LOG UseReservedNumber / Success / Info
│   │
│   └── RETURN success to Canvas App
```

---

## Business rules enforced by this flow

### 1. Reserved number must exist

The item with `reservedNumberId` must be present in RezervisaniBrojevi.
A 404 from SharePoint Get item means the number was already deleted (used or removed by an admin).
Error code: `RESERVED_NUMBER_NOT_FOUND`

### 2. Year must match — two-level check

**Level 1 — Item year vs requestedYear:**
The year stored on the reserved number item must equal the `requestedYear` input.
A mismatch means the user is trying to use a number reserved for a different year.
Error code: `RESERVED_NUMBER_YEAR_MISMATCH`

**Level 2 — requestedYear vs active year in App Config:**
The `requestedYear` must equal the active year configured in App Config.
This prevents registering documents into future years or re-opening closed years.
Error code: `YEAR_NOT_ACTIVE`

Both checks must pass. The item-year check (Level 1) runs first because it requires
only one SharePoint read. The App Config check (Level 2) runs second.

### 3. Number must not already be in use

Even if the reserved number item exists in RezervisaniBrojevi, a document in Svi predmeti
might already carry the same DelovodniBroj. This would indicate a data consistency issue
(e.g. the reserved number was not deleted after a previous successful use).

The flow queries Svi predmeti for any item with the same DelovodniBroj value.
If found, it returns `RESERVED_NUMBER_ALREADY_USED` with Critical severity.
An administrator must investigate this case — it should never occur under normal operation.

### 4. Any document type is valid

Reserved numbers are not tied to a document type. Any user who can register documents
can use any reserved number. No document-type validation is performed.

### 5. All users see all reserved numbers

Authorization to use a reserved number is not checked within this flow.
Access control is handled at the Canvas App level (the scrNoviPredmet screen shows all
reserved numbers in RezervisaniBrojevi to all users who can register documents).

### 6. Reserved number cannot be manually deleted

Users do not have Delete permissions on RezervisaniBrojevi. Only the service account
(via CF_DocCentralV3_CreateDocument) can delete reserved number items, and only after
successful document creation. This rule is enforced by SharePoint list permissions,
not by this flow.

---

## What happens on document creation failure after validation

If CF_DocCentralV3_UseReservedNumber returns success but CF_DocCentralV3_CreateDocument
subsequently fails (e.g. Svi predmeti Create item action fails or file upload fails):

- The reserved number item in RezervisaniBrojevi is NOT deleted.
- The delovodniBroj is NOT recorded in any Svi predmeti item.
- The user can retry document registration with the same reserved number.
- No manual cleanup is needed.

This is the correct behaviour — the reserved number is only consumed on a fully
successful end-to-end document creation.

---

## Edge case: user submits the same form twice quickly

If a user submits the same reserved number twice in quick succession (e.g. double-click):

1. First call to CF_DocCentralV3_UseReservedNumber passes (no document exists yet).
2. First call to CF_DocCentralV3_CreateDocument creates the Svi predmeti item and deletes the reserved number.
3. Second call to CF_DocCentralV3_UseReservedNumber:
   - Either: Get item returns 404 (already deleted) → RESERVED_NUMBER_NOT_FOUND.
   - Or: Svi predmeti duplicate check finds the item created by the first call → RESERVED_NUMBER_ALREADY_USED.

In both cases the second call fails safely. No duplicate document is created.

---

## Audit log sequence for a successful consumption

| Flow | EventType | Status | Severity | When |
|---|---|---|---|---|
| CF_DocCentralV3_UseReservedNumber | UseReservedNumber | Started | Info | Flow begins |
| CF_DocCentralV3_UseReservedNumber | UseReservedNumber | Success | Info | Validation passes (number not yet deleted) |
| CF_DocCentralV3_CreateDocument | UseReservedNumber | Success | Info | After reservation item deleted |

All three entries share the same correlationId passed from CF_DocCentralV3_CreateDocument.

---

## Audit log sequence for a validation failure

| Flow | EventType | Status | Severity | errorCode |
|---|---|---|---|---|
| CF_DocCentralV3_UseReservedNumber | UseReservedNumber | Started | Info | — |
| CF_DocCentralV3_UseReservedNumber | UseReservedNumber | Failed | Error or Critical | RESERVED_NUMBER_NOT_FOUND / RESERVED_NUMBER_YEAR_MISMATCH / YEAR_NOT_ACTIVE / RESERVED_NUMBER_ALREADY_USED |

Severity mapping:
- RESERVED_NUMBER_NOT_FOUND → Error (unexpected — item should exist)
- RESERVED_NUMBER_YEAR_MISMATCH → Warning (user/data mismatch, not a system error)
- YEAR_NOT_ACTIVE → Warning (configuration issue or UI mismatch)
- RESERVED_NUMBER_ALREADY_USED → Critical (data integrity issue — requires admin investigation)
