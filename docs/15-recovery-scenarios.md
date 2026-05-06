# 15 - Recovery Scenarios

Last updated: 2026-05-06

---

## Recovery principle

Any partially failed transaction that touches `DelovodniBroj` must be detected, logged, and manually resolved before new registrations can continue. Registry number continuity is a hard business requirement. A gap or a duplicate cannot be corrected after the fact without legal/administrative impact.

When in doubt: block first, investigate, then unblock. Do not attempt to auto-heal number assignment failures without an admin confirming the transaction state.

---

## How recovery is triggered

Recovery is available through two channels:

1. **Canvas app recovery panel** — visible only to users with Admin or Consultant role. Shows all `Svi predmeti` items where `TechnicalStatus = FailedNeedsRecovery` or `TechnicalStatus = MetadataSyncFailed`. Provides a per-item "Recover" button that calls `CF_DocCentral_RecoverTransaction`.

2. **`CF_DocCentral_RecoverTransaction` flow** — the recovery flow accepts a `Svi predmeti` item ID and performs the appropriate repair based on the detected state. It does not guess; it reads the current state and acts on what it finds.

---

## How a FailedNeedsRecovery item blocks registrations

Every execution of `CF_DocCentral_AssignRegistryNumber` includes the following check as step 2 (after acquiring the lock, before reading the counter):

```
Get items from Svi predmeti
Filter: TechnicalStatus eq 'FailedNeedsRecovery'
Top: 1
If count > 0:
    Release lock
    Return: { success: false, code: "BLOCKED_RECOVERY_REQUIRED", message: "..." }
```

This makes the block automatic and does not require Canvas app logic to enforce it (the flow does it). Canvas app adds a second layer of warning UI, but the flow is the authoritative gate.

---

## REC-001: Svi predmeti item exists without DelovodniBroj

### Trigger condition

A `Svi predmeti` item was created with `TechnicalStatus = NumberAssignmentInProgress` but the subsequent write of `DelovodniBroj` failed. The item exists without a number.

### State

| Data store | State |
|---|---|
| `Svi predmeti` | Item exists; `DelovodniBroj` is blank; `TechnicalStatus = FailedNeedsRecovery` |
| `Dokumenta` | File may exist but `DelovodniBroj` is blank |
| `NumberLocks` | Lock item with `Status = Failed` and `ErrorCode = ASSIGN_FAILED` |
| `AppConfig` counter | Not yet updated (failure occurred before step 10) |

### Impact

All new registrations are blocked.

### Recovery flow actions

```
1. Read the affected Svi predmeti item
2. Confirm DelovodniBroj is indeed blank
3. Read AppConfig DelovodneKnjige → get ActiveYear and CurrentCounter
4. Query Svi predmeti max(DelovodniBroj) for this book/year
5. Determine the correct next number (same logic as normal assignment, step 4)
6. Verify the number is not already assigned (extra safety check)
7. Write DelovodniBroj to Svi predmeti
8. Write DelovodniBroj to Dokumenta if file exists
9. Update AppConfig CurrentCounter
10. Set TechnicalStatus = Completed on Svi predmeti
11. Update NumberLocks Status = Released (or create Released record if lock item was deleted)
12. Write AuditLog: RecoveryCompleted, include old state and new DelovodniBroj
```

### What admin must confirm before triggering recovery

- The file was intended to be registered (not an accidental upload)
- The document metadata is correct
- No other item in `Svi predmeti` already holds the number that recovery will assign

### Outcome

`TechnicalStatus = Completed`, `DelovodniBroj` assigned, registrations unblocked.

---

## REC-002: DelovodniBroj in Svi predmeti but not in Dokumenta

### Trigger condition

Number assignment succeeded in `Svi predmeti` (step 8 of transaction) but the metadata sync to `Dokumenta` (step 9) failed. This is a lower-severity state because the source of truth (`Svi predmeti`) is consistent. The number sequence is not at risk.

### State

| Data store | State |
|---|---|
| `Svi predmeti` | Item exists; `DelovodniBroj` assigned; `TechnicalStatus = MetadataSyncFailed` |
| `Dokumenta` | File exists; `DelovodniBroj` blank on file metadata |
| `NumberLocks` | Lock may be Released (assignment succeeded) |
| `AppConfig` counter | May or may not be updated depending on failure timing |

### Impact

New registrations are **not** blocked (this is `MetadataSyncFailed`, not `FailedNeedsRecovery`). However, the document will not appear correctly in views that filter by `DelovodniBroj` on `Dokumenta`.

### Recovery flow actions

```
1. Read the affected Svi predmeti item → get DelovodniBroj
2. Find the corresponding Dokumenta item (by RelatedItemId or document name match)
3. Write DelovodniBroj to Dokumenta item metadata
4. Set TechnicalStatus = Completed on Svi predmeti
5. Write AuditLog: RecoveryCompleted
```

### Outcome

`Dokumenta` metadata synchronized. `TechnicalStatus = Completed`.

---

## REC-003: Failed deletion of used reserved number

### Trigger condition

A reserved number was successfully used (written to both `Svi predmeti` and `Dokumenta`), but the delete action on `Rezervisani brojevi` (step 11 of transaction) failed. The number is now assigned and also still present in the reserved list, creating a risk of future duplicate use.

### State

| Data store | State |
|---|---|
| `Svi predmeti` | Item exists with correct `DelovodniBroj`; `TechnicalStatus = Completed` or `MetadataSyncFailed` |
| `Dokumenta` | Metadata correct |
| `Rezervisani brojevi` | Reserved number item still exists (should have been deleted) |
| `NumberLocks` | Lock released |

### Impact

New registrations are **not** blocked. Risk: if the reserved number entry is not deleted, it could be selected again for a future reserved registration, producing a duplicate.

### Recovery flow actions

```
1. Identify the reserved number entry in Rezervisani brojevi that matches the used DelovodniBroj
2. Verify the number is already assigned in Svi predmeti (confirm it was used)
3. Delete the Rezervisani brojevi item
4. Write AuditLog: RecoveryCompleted, reserved number {N} deleted
```

### Automatic detection

The `CF_DocCentral_AssignRegistryNumber` flow (step 5) checks whether the candidate next number is in `Rezervisani brojevi` when not on the reserved path. If it finds a reserved number that matches an already-assigned `DelovodniBroj`, this is the REC-003 state. The flow should: log a warning, delete the stale reserved entry, and continue assignment with the correct next number.

---

## REC-004: Archive book PDF created but not registered

### Trigger condition

The archive book generation flow produced a PDF and uploaded it to `Dokumenta`, but the subsequent write to `Svi predmeti` (number assignment + metadata) failed.

### State

| Data store | State |
|---|---|
| `Dokumenta` | Archive book file exists without `DelovodniBroj` |
| `Svi predmeti` | No item for the archive book (or item with `TechnicalStatus = FailedNeedsRecovery`) |
| AppConfig | Archive book flag for the year may or may not be set |

### Impact

All registrations blocked if `FailedNeedsRecovery` item exists. Duplicate archive book generation is also possible if the archive book flag was not set before the failure.

### Recovery flow actions

```
1. Check if Svi predmeti already has an archive book item for this year (duplicate check)
2. If no item exists: create Svi predmeti item and assign DelovodniBroj through normal assignment path
3. If item exists with FailedNeedsRecovery: complete it via REC-001 path
4. Set archive book generated flag in AppConfig
5. Write AuditLog: RecoveryCompleted
```

### Guard against duplicate generation

The archive book generation flow must check for:
- An existing `Svi predmeti` item with archive book document type for the year → block generation
- An existing file in `Dokumenta` matching the archive book name pattern for the year → block generation and trigger recovery check

---

## REC-005: AppConfig CurrentCounter update failed

### Trigger condition

Number assignment to `Svi predmeti` and `Dokumenta` succeeded (steps 8 and 9), but the AppConfig counter update (step 10) failed. The counter is now lower than the highest assigned number.

### State

| Data store | State |
|---|---|
| `Svi predmeti` | Item with correct `DelovodniBroj`; `TechnicalStatus = Completed` |
| `AppConfig` | `CurrentCounter` is stale (lower than actual max assigned number) |

### Impact

New registrations are **not** blocked. The next assignment flow will detect the mismatch at step 4 (by querying `max(DelovodniBroj)`) and self-correct. The stale counter does not produce duplicates or gaps because the flow never relies solely on the counter.

### Recovery flow actions

```
1. Query Svi predmeti max(DelovodniBroj) for this book/year
2. Update AppConfig CurrentCounter to match
3. Write AuditLog: Info — counter corrected
```

This can be done automatically by the next successful registration flow run, or triggered manually if needed.

---

## REC-006: Temporary file cleanup failed

### Trigger condition

The `CF_DocCentral_EmailDocumentsCleanup` or `CF_DocCentral_CleanupAuditLog` flow failed during cleanup of temporary or old files. No business data is affected.

### State

No business data at risk. Stale files or log entries remain.

### Impact

No functional impact. Disk/storage quota consumption increases until cleanup succeeds.

### Recovery

The next scheduled run of the cleanup flow will reattempt deletion. No manual recovery is needed unless the list/library has grown to a size that causes SharePoint throttling, in which case the cleanup interval should be shortened and batch size reduced.

---

## REC-007: Lock expired — transaction state unknown

### Trigger condition

A `NumberLocks` item exists with `Status = Active` and `ExpiresAt < utcNow()`. The flow that held the lock did not release it, which means the flow was interrupted (timeout, platform error, connection drop).

### State

Unknown. The interrupted flow may have:
- Not written anything (safe to retry)
- Written to `Svi predmeti` only (REC-001 state)
- Written to both `Svi predmeti` and `Dokumenta` but not released the lock

### Impact

All new registrations are blocked because any flow encountering the expired lock must treat it as a potential REC-001 state.

### Recovery flow actions

```
1. Read the expired NumberLocks item → get RelatedItemId
2. Query Svi predmeti item with ID = RelatedItemId
3. Evaluate TechnicalStatus:
   └─ TechnicalStatus = Completed → lock was released late; mark lock as Released; unblock
   └─ TechnicalStatus = NumberAssignmentInProgress or FailedNeedsRecovery → execute REC-001
   └─ TechnicalStatus = MetadataSyncFailed → execute REC-002
   └─ Item does not exist → flow failed before creating the item; mark lock as Released; unblock
4. Update NumberLocks Status = Expired (or Released if confirmed safe)
5. Write AuditLog: LockExpiredDetected, action taken
```

---

## Recovery flow — CF_DocCentral_RecoverTransaction

This flow does not exist yet. A manual implementation guide must be written for it. The flow is the backend for the admin recovery panel in the Canvas app.

### Required inputs

| Parameter | Type | Notes |
|---|---|---|
| `itemId` | number | `Svi predmeti` item ID of the affected registration |
| `recoveryAction` | text | `ASSIGN_NUMBER`, `SYNC_DOKUMENTA`, `DELETE_RESERVED`, `RELEASE_LOCK` |
| `requestedByEmail` | text | Admin/consultant triggering recovery |

### Required outputs

| Field | Type | Notes |
|---|---|---|
| `success` | boolean | |
| `code` | text | Machine-readable result code |
| `message` | text | User-friendly message |
| `assignedNumber` | number | Assigned number if applicable |
| `technicalDetail` | text | For AuditLog/support |

### Required connections

- Service account SharePoint connection reference
- AppConfig write access

### Critical constraints

- The recovery flow must acquire a `NumberLocks` lock before assigning a number, using the same create-then-validate pattern as the main assignment flow
- The recovery flow must verify the item is still in a recoverable state before acting (another admin may have already recovered it)
- The recovery flow must write an AuditLog entry for every action taken

---

## Admin recovery panel — Canvas app concept

The recovery panel is a section visible only to Admin and Consultant roles. It must not be visible or accessible to any other user.

### Panel contents

1. **Banner** — shown on registration screen to all users when `FailedNeedsRecovery` items exist:
   > "Document registration is currently suspended. An administrator must resolve pending issues before new registrations can proceed."

2. **Recovery list** (Admin/Consultant only) — a gallery showing all `Svi predmeti` items where `TechnicalStatus` is `FailedNeedsRecovery` or `MetadataSyncFailed`. Columns shown:
   - ID
   - Document name
   - TechnicalStatus
   - Created date
   - ErrorCode (from the related NumberLocks item if available)

3. **Recover button** (per item) — calls `CF_DocCentral_RecoverTransaction` with `itemId` and the appropriate `recoveryAction`. Wrapped in `IfError`. Shows result message.

4. **Refresh button** — reloads the recovery list after a recovery action.

The panel is located on a dedicated admin screen or as a section of `scrSettings` / `scrHome` visible only when the user has the appropriate role and `FailedNeedsRecovery` items exist.

---

## Test matrix — registry number locking and recovery

### Normal path

| ID | Scenario | Preconditions | Expected result |
|---|---|---|---|
| NL-001 | Single user registers document | No active locks, no failed items | Number assigned sequentially, lock released, TechnicalStatus=Completed |
| NL-002 | Register with reserved number | Reserved number exists in Rezervisani brojevi | Reserved number used, deleted from list, TechnicalStatus=Completed |
| NL-003 | Register: AppConfig counter is stale | Counter = 5, max in Svi predmeti = 7 | Next number = 8, counter corrected, AuditLog warning written |
| NL-004 | Register: AppConfig counter is ahead | Counter = 10, max in Svi predmeti = 8 | Next number = 10 (use counter), TechnicalStatus=Completed |

### Concurrency

| ID | Scenario | Preconditions | Expected result |
|---|---|---|---|
| NL-010 | 2 concurrent registrations, same registry book | No active locks | One succeeds immediately; second detects lock collision, returns "system busy" error; Canvas app shows retry message; after first completes, second succeeds with next sequential number |
| NL-011 | 10 concurrent registrations | No active locks | 10 sequential unique numbers assigned, no duplicates, no skips; order may vary but sequence is gapless |
| NL-012 | 2 concurrent registrations, different registry books | No active locks | Both succeed in parallel; different LockKey values do not block each other |

### Lock expiry

| ID | Scenario | Preconditions | Expected result |
|---|---|---|---|
| NL-020 | Flow holds lock and times out | NumberLocks item with Status=Active, ExpiresAt in the past | New registration attempt detects expired lock, returns recovery required error, new number NOT assigned |
| NL-021 | Expired lock, underlying item has TechnicalStatus=Completed | | Recovery flow marks lock as Released, unblocks registrations |
| NL-022 | Expired lock, underlying item has TechnicalStatus=NumberAssignmentInProgress | | Recovery flow executes REC-001 path, assigns number, unblocks |
| NL-023 | Expired lock, no Svi predmeti item found | | Recovery flow marks lock as Released, unblocks registrations |

### FailedNeedsRecovery blocking

| ID | Scenario | Preconditions | Expected result |
|---|---|---|---|
| NL-030 | Registration attempt while FailedNeedsRecovery exists | Svi predmeti item with TechnicalStatus=FailedNeedsRecovery | Flow returns BLOCKED_RECOVERY_REQUIRED; Canvas shows blocking banner |
| NL-031 | Ordinary user sees blocking banner | Same as NL-030 | No registration form accessible, no bypass available |
| NL-032 | Admin sees recovery panel | Same as NL-030 | Recovery panel visible, affected items listed |
| NL-033 | Admin triggers recovery for REC-001 | Item without DelovodniBroj | Number assigned, TechnicalStatus=Completed, banner disappears, registration unblocked |

### Recovery scenarios

| ID | Scenario | Preconditions | Expected result |
|---|---|---|---|
| NL-040 | REC-001: item without DelovodniBroj | TechnicalStatus=FailedNeedsRecovery, DelovodniBroj blank | Recovery assigns correct next number; both Svi predmeti and Dokumenta updated; AuditLog entry |
| NL-041 | REC-002: DelovodniBroj in Svi predmeti, not in Dokumenta | TechnicalStatus=MetadataSyncFailed | Recovery updates Dokumenta metadata; TechnicalStatus=Completed; new registrations not blocked |
| NL-042 | REC-003: Reserved number not deleted after use | Reserved number still in Rezervisani brojevi, already assigned in Svi predmeti | Recovery deletes reserved entry; AuditLog entry |
| NL-043 | REC-005: AppConfig counter stale | Counter behind max assigned | Next registration self-corrects counter; or manual recovery updates counter |
| NL-044 | REC-007: Expired lock | Lock Status=Active, ExpiresAt < now | Recovery identifies state, takes correct action (one of NL-021/022/023 paths) |

### Safeguards

| ID | Scenario | Expected result |
|---|---|---|
| NL-050 | Flow attempts to assign a number already in Svi predmeti | Duplicate detected at step 5 → FailedNeedsRecovery, Critical AuditLog, no number written |
| NL-051 | Flow attempts to use a reserved number not in Rezervisani brojevi | Reserved path: error returned, no number assigned |
| NL-052 | DelovodniBroj unique constraint violated at SharePoint level | SharePoint returns 409 conflict; flow Catch scope handles it; FailedNeedsRecovery set |
| NL-053 | Recovery flow called on already-recovered item | TechnicalStatus=Completed detected; recovery returns "already resolved" message; no changes |
| NL-054 | Recovery flow called by user without admin/consultant role | Canvas app blocks the call; flow validates role if possible |
