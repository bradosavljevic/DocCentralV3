# 08 - Registry Number Rules

## Critical rule

`DelovodniBroj` must never be duplicated and must never be skipped.

## Sequential assignment

If number 3 fails, system must not continue with 4 until 3 is resolved.

## Reserved numbers

Reserved numbers are the only allowed exception to normal sequence.

Rules:
- stored in `Rezervisani brojevi`
- valid only for active year
- may be used for any document type
- must be used before year closing
- must not be manually deleted before use
- after successful use, delete from `Rezervisani brojevi`

## Lock mechanism

Use `NumberLocks` / `ZakljucavanjeBrojeva`.

- lock duration: 5 minutes
- unique LockKey per registry book/year
- expired lock requires recovery validation
- expired lock does not automatically allow continuation

## FailedNeedsRecovery

If number assignment enters `FailedNeedsRecovery`:
- block all new registrations
- do not assign next number
- ordinary users cannot resolve it
- admin/consultant or service recovery flow must fix it
- app should provide recovery button for authorized user

## Transaction order

1. Acquire NumberLock.
2. Check unresolved FailedNeedsRecovery.
3. Determine active registry book/year.
4. Determine next number.
5. Verify next number is not already assigned.
6. Verify next number is not reserved unless reserved path is used.
7. Update `Svi predmeti`.
8. Update `Dokumenta`.
9. Update AppConfig counter.
10. Delete used reserved number if applicable.
11. Write AuditLog success.
12. Release lock.
