# 08 - Registry Number Rules

Last updated: 2026-05-06

---

## Core invariant

`DelovodniBroj` must never be duplicated and must never be skipped.

These two rules are non-negotiable. Every design decision in number assignment must be evaluated against them. When in doubt, block registration rather than risk a duplicate or a gap.

---

## Sequential assignment rule

Numbers are assigned sequentially per registry book per year. The next number is always `max(assigned numbers in this book/year) + 1`.

If number N fails during assignment, the system must **not** continue with N+1 until N is resolved. A gap is as bad as a duplicate.

### Why this matters

If a failure produces a partially written transaction (item in `Svi predmeti` without a number), the next flow run must not assume the number was never used. It must detect the incomplete state and block until recovery confirms whether the number was assigned or not.

---

## Source of truth

`Svi predmeti` is the source of truth for assigned `DelovodniBroj` values.

- `Dokumenta` metadata is derived from `Svi predmeti` and must be kept in sync
- AppConfig `DelovodneKnjige` counter is a performance cache; it is updated after the write to `Svi predmeti` succeeds, not before
- If `Svi predmeti` and AppConfig counter disagree, `Svi predmeti` max number wins

---

## Reserved numbers

Reserved numbers are the only allowed exception to normal sequence rules.

| Rule | Detail |
|---|---|
| Storage | `Rezervisani brojevi` list |
| Scope | Valid only for the active year |
| Document type | May be used for any `TipDokumenta` |
| Year closing | All reserved numbers must be used or manually cancelled before year closing is allowed |
| Manual deletion | Reserved numbers must not be deleted before use unless an admin explicitly cancels the reservation with an AuditLog entry |
| Post-use deletion | A used reserved number must be deleted from `Rezervisani brojevi` only after **both** `Svi predmeti` and `Dokumenta` are successfully updated |

---

## Lock mechanism

All number assignment must go through the `NumberLocks` / `ZakljucavanjeBrojeva` list.

See `sharepoint-guides/NumberLocks.md` for list specification.

### Lock acquisition — safe pattern

The flow must use the **create-then-validate** pattern:

1. Attempt to create a `NumberLocks` item with the `LockKey` for this registry book + year
2. If the create succeeds → lock is acquired (because `LockKey` has a unique constraint, no other flow can also create an item with the same key)
3. If the create fails with a conflict/duplicate error → another flow holds the lock → return a "system busy, try again" error to Canvas — do **not** proceed

The alternative "check for existing lock, then create" pattern is **unsafe** because two flows can both see no lock, both decide to create, and both create a lock simultaneously. Do not implement this pattern.

### Lock duration

- Duration: **5 minutes**
- `ExpiresAt = LockedAt + 5 minutes` (UTC)
- Lock must be released (Status = Released) in the Finally scope of the flow under all success and non-recovery failure conditions

### Expired lock

An expired lock (`Status = Active` AND `ExpiresAt < utcNow()`) means the previous flow run that held the lock did not complete cleanly. The transaction state is unknown.

Rules:
- An expired lock does **not** automatically grant a new flow run permission to proceed
- The flow that encounters an expired lock must: log an AuditLog warning, return an error to Canvas, and require an admin to investigate before assignment resumes
- Only the recovery flow may advance an expired lock to `Status = Expired` after confirming what happened to the related `Svi predmeti` item

---

## TechnicalStatus lifecycle

`TechnicalStatus` is a column on `Svi predmeti` that tracks the technical state of a registration transaction separately from the business status (`Stanje`).

| Value | Meaning |
|---|---|
| `None` | No active transaction; item is fully settled |
| `DocumentCreationInProgress` | File is being uploaded to `Dokumenta`; `Svi predmeti` item not yet created |
| `NumberAssignmentInProgress` | `Svi predmeti` item exists; number assignment is in progress |
| `Completed` | Transaction fully complete; all writes successful |
| `FailedNeedsRecovery` | Transaction failed with data integrity risk; admin action required |
| `MetadataSyncFailed` | `Svi predmeti` has number; `Dokumenta` metadata sync failed (lower severity) |

A `Svi predmeti` item may exist temporarily without a `DelovodniBroj` while `TechnicalStatus = NumberAssignmentInProgress`. This is expected and must not trigger alerts. The item must not be treated as registered until `TechnicalStatus = Completed`.

---

## FailedNeedsRecovery — blocking rule

When any `Svi predmeti` item has `TechnicalStatus = FailedNeedsRecovery`:

1. **Block all new registrations.** The `CF_DocCentral_AssignRegistryNumber` flow must check for `FailedNeedsRecovery` items at step 2 of the transaction order (after acquiring the lock, before determining the next number). If any are found, the flow must return an error immediately.
2. **Canvas app must surface the block.** The registration screen must show a clear message to ordinary users explaining that registration is suspended.
3. **Canvas app must provide a recovery entry point** visible only to users with admin or consultant role. This entry point shows the affected items and allows triggering `CF_DocCentral_RecoverTransaction`.
4. **Ordinary users cannot skip or override the block.** There is no bypass.

---

## Required changes to Svi predmeti

The following columns must exist on `Svi predmeti` for the locking and recovery mechanism to work:

| Display name | Suggested internal name | Type | Notes |
|---|---|---|---|
| TechnicalStatus | TechnicalStatus | Choice | Values: None, DocumentCreationInProgress, NumberAssignmentInProgress, Completed, FailedNeedsRecovery, MetadataSyncFailed |
| DelovodniBroj | DelovodniBroj | Single line text | **Indexed + Unique required** |

`DelovodniBroj` must have:
- **Enforce unique values = Yes** — last line of defence against duplicates if flow logic has a bug
- Index added via List Settings → Indexed columns

---

## Required changes to Dokumenta

The following columns must exist on `Dokumenta`:

| Display name | Suggested internal name | Type | Notes |
|---|---|---|---|
| DelovodniBroj | DelovodniBroj | Single line text | Indexed; does not need unique constraint (multiple files can share a number via attachments) |
| Attachment | Attachment | Yes/No | `false` = main document, `true` = attachment |
| TechnicalStatus | TechnicalStatus | Choice | Same values as Svi predmeti; used during sync validation |

---

## Required changes to AppConfig — DelovodneKnjige

The `DelovodneKnjige` AppConfig key must include the following fields in each registry book JSON object:

| Field | Type | Notes |
|---|---|---|
| ID | GUID string | Stable identifier for this registry book |
| Naziv | string | Display name |
| Key | string | Short key used as `RegistryBookKey` in LockKey format |
| ActiveYear | number | Currently active year for this book |
| CurrentCounter | number | Last successfully assigned number for the active year |
| IsActive | boolean | Whether this book is currently accepting new registrations |
| ClosedYears | array of numbers | Years that have been closed for this book |

`CurrentCounter` is a performance cache and is updated **after** the number write to `Svi predmeti` succeeds. If the two values disagree, `max(DelovodniBroj in Svi predmeti for this book/year)` is the authoritative next-number base.

---

## Required AuditLog entries

The following events must produce AuditLog entries as part of the locking and number assignment mechanism:

| EventType | Severity | When |
|---|---|---|
| `NumberAssigned` | Info | Successful number assignment |
| `NumberAssignmentFailed` | Critical | Any failure that sets TechnicalStatus = FailedNeedsRecovery |
| `NumberAssignmentBlocked` | Warning | Flow rejected because FailedNeedsRecovery exists |
| `LockAcquired` | Info | Lock successfully created (optional, for diagnostics) |
| `LockReleased` | Info | Lock successfully released (optional) |
| `LockExpiredDetected` | Warning | Flow detected an expired active lock |
| `LockCollisionDetected` | Warning | Flow detected another active lock for the same key |
| `ReservedNumberUsed` | Info | Reserved number successfully used and deleted |
| `ReservedNumberDeletionFailed` | Warning | Number used but deletion from Rezervisani brojevi failed |
| `RecoveryStarted` | Info | Admin triggered recovery flow |
| `RecoveryCompleted` | Info | Recovery flow completed successfully |
| `RecoveryFailed` | Critical | Recovery flow itself failed |
| `MetadataSyncFailed` | Warning | Svi predmeti updated but Dokumenta sync failed |

---

## Transaction order — complete sequence

The following order must be followed for every normal (non-reserved-number) registration. Do not reorder steps.

```
1.  Acquire lock
    └─ Create NumberLocks item (Status=Active, LockKey, ExpiresAt=now+5min)
    └─ If create fails (duplicate key conflict) → return "system busy" error
    └─ If existing Active lock with same key and ExpiresAt < now → expired lock detected → return recovery required error

2.  Check FailedNeedsRecovery
    └─ Query Svi predmeti: any item where TechnicalStatus eq 'FailedNeedsRecovery'
    └─ If yes → update NumberLocks Status=Released → return blocked error

3.  Read AppConfig DelovodneKnjige
    └─ Get active year and CurrentCounter for this registry book

4.  Determine next number
    └─ Query Svi predmeti: max(DelovodniBroj) for this book + year
    └─ next = max + 1
    └─ If AppConfig counter > max → use AppConfig counter (forward-safe)
    └─ If AppConfig counter < max → AppConfig is stale; use max + 1 and log warning

5.  Verify number is not already assigned
    └─ Query Svi predmeti: filter DelovodniBroj eq next
    └─ If found → duplicate detected → FailedNeedsRecovery → return critical error

6.  Verify number is not in Rezervisani brojevi (unless reserved path)
    └─ Query Rezervisani brojevi: filter RezervisaniBroj eq next
    └─ If found and not using reserved path → skip to next number? No → block and return error
       (Operator must explicitly cancel the reservation before sequence can use that number)

7.  Set TechnicalStatus = NumberAssignmentInProgress on Svi predmeti item

8.  Write DelovodniBroj to Svi predmeti
    └─ If write fails → set TechnicalStatus = FailedNeedsRecovery
                     → update NumberLocks Status=Failed
                     → write AuditLog Critical
                     → return error

9.  Write DelovodniBroj and metadata to Dokumenta
    └─ If write fails → Svi predmeti has number, Dokumenta does not
                     → set TechnicalStatus = MetadataSyncFailed on Svi predmeti
                     → write AuditLog Warning (REC-002 scenario)
                     → continue (MetadataSyncFailed is lower severity than FailedNeedsRecovery)

10. Update AppConfig CurrentCounter
    └─ If write fails → AuditLog warning (REC-005 scenario)
                     → counter mismatch will be corrected on next assignment via step 4 max() query
                     → do not set FailedNeedsRecovery for counter-only failures

11. If reserved number path: delete used reserved number from Rezervisani brojevi
    └─ If delete fails → AuditLog warning (REC-003 scenario)
                      → set ErrorCode on NumberLocks
                      → recovery flow will retry deletion

12. Set TechnicalStatus = Completed on Svi predmeti

13. Write AuditLog success (NumberAssigned)

14. Update NumberLocks Status = Released

15. Return success response to Canvas
```

### Transaction order — reserved number path

Steps are the same except:
- Step 6 is skipped (the number is expected to be in `Rezervisani brojevi`)
- Step 6 is replaced with: verify the specific reserved number exists in `Rezervisani brojevi` and belongs to the active year
- Step 11 is mandatory and uses the specific reserved number ID

---

## Blocking check in Canvas app

Before the registration form is submitted, the Canvas app must call the `CF_DocCentral_AssignRegistryNumber` flow (or a lightweight check flow) that returns a `blocked` flag if any `FailedNeedsRecovery` item exists. The form must not allow submission while blocked.

Alternatively, the Canvas app can load a count of `FailedNeedsRecovery` items from a read flow on screen load and show a banner if count > 0.

The block check must not be bypassable from the Canvas app UI by any user other than admin/consultant, who see the recovery button instead of the blocked banner.
