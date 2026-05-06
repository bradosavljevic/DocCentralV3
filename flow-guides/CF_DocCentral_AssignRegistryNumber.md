# Flow Guide: CF_DocCentral_AssignRegistryNumber

Last updated: 2026-05-06 (final pre-implementation revision)

---

## Purpose

Assigns the next sequential registry number to a document that has already been created as a `Svi predmeti` item and a `Dokumenta` file. Uses `NumberLocks` to prevent concurrent assignment. Enforces strict no-duplicate and no-skip rules. Blocks if any `FailedNeedsRecovery`, `MetadataSyncFailed`, or `CounterSyncFailed` item exists. Supports a reserved number path.

This flow is a **child flow** called by `CF_DocCentral_CreateDocumentTransaction`. It is not called directly from Canvas app.

---

## Design principles

1. **Counter is authoritative.** The next number is always `CurrentCounter + 1`. If `Svi predmeti` max disagrees with the counter in any direction that cannot be explained, the flow stops and returns `COUNTER_OUT_OF_SYNC` or `COUNTER_AHEAD_NEEDS_REVIEW`.

2. **No skipping under any circumstances.** If the next sequential number is reserved and `useReservedNumber = false`, the flow returns `NEXT_NUMBER_IS_RESERVED` and blocks automatic assignment. It never increments past a reserved number to find the next free one.

3. **MetadataSyncFailed and CounterSyncFailed block new registrations.** Both states, together with `FailedNeedsRecovery`, are treated as blocking conditions. None of them may be bypassed by a new registration attempt.

4. **AppConfig counter failure blocks new registrations.** If the AppConfig counter update fails after a number has been assigned, `TechnicalStatus = CounterSyncFailed` is set. The next flow run cannot safely determine the correct next number until the counter is repaired.

5. **No `Terminate(Cancelled)` for expected business outcomes.** All paths use output variables. The flow always ends at the Respond action with a fully populated response, regardless of which path was taken.

6. **Lock state reflects outcome precisely.** Lock is set to `Released` when no persistent data change was made or when the number was safely written. Lock is set to `Failed` only when data was partially written and recovery is required.

7. **`numberAssigned` distinguishes three states:** number not assigned, number assigned but pending recovery, full success.

---

## Trigger

**Type:** Child flow — called from `CF_DocCentral_CreateDocumentTransaction` via the **Run a Child Flow** action.

For standalone testing: **Manually trigger a flow** (not Power Apps V2).

---

## Connections

All SharePoint actions must use `CR_DocCentralV3_SharePoint` (service account connection reference). No user-delegated connection.

---

## Environment variables used

| Variable | Purpose |
|---|---|
| `EV_DocCentralV3_lstSviPredmeti` | Site-relative URL of `Svi predmeti` list |
| `EV_DocCentralV3_lstAppConfig` | Site-relative URL of `AppConfig` list |
| `EV_DocCentralV3_docDokumenti` | Site-relative URL of `Dokumenta` library |
| `EV_DocCentralV3_lstNumberLocks` | Site-relative URL of `NumberLocks` list — **must be created; developer returns variable name** |
| `EV_DocCentralV3_lstRezervBrojevi` | Site-relative URL of `Rezervisani brojevi` list |
| `EV_DocCentralV3_SharePointSite` | Root SharePoint site URL |

---

## SharePoint column model for DelovodniBroj

Registry numbers are stored in four separate columns on both `Svi predmeti` and `Dokumenta`. A single formatted text column is insufficient for sequence queries, year filtering, and uniqueness enforcement.

| Display name | Suggested internal name | Type | Unique | Indexed | Example |
|---|---|---|---|---|---|
| DelovodniBrojNumber | DelovodniBrojNumber | Number (integer) | **No** | **Yes** | `42` |
| DelovodniBrojText | DelovodniBrojText | Single line text | **Yes** | **Yes** | `"DK01-42/2026"` |
| DelovodniBrojGodina | DelovodniBrojGodina | Number (integer) | No | Yes | `2026` |
| DelovodnaKnjigaKey | DelovodnaKnjigaKey | Single line text | No | Yes | `"DK01"` |

### Why DelovodniBrojNumber is NOT unique

`DelovodniBrojNumber` repeats across years (number 1 exists in 2025 and again in 2026) and may repeat across registry books. A unique constraint on this column alone would prevent valid records. Uniqueness is enforced at the composite level: `(DelovodniBrojNumber + DelovodnaKnjigaKey + DelovodniBrojGodina)` is the logical composite key. This is enforced by the duplicate check in step 5 and by the unique constraint on `DelovodniBrojText`, which encodes all three components.

### DelovodniBrojText format

```
{DelovodnaKnjigaKey}-{DelovodniBrojNumber}/{DelovodniBrojGodina}
```

Example: `DK01-42/2026`

`DelovodniBrojText` carries both the **Indexed = Yes** and **Enforce unique values = Yes** settings. It is the hard uniqueness safety net at the SharePoint level.

---

## TechnicalStatus values — Svi predmeti

All values used by this flow:

| Value | Blocks new registrations | Set by |
|---|---|---|
| `None` | No | Initial state or manual reset |
| `DocumentCreationInProgress` | No | CreateDocumentTransaction flow |
| `NumberAssignmentInProgress` | No | Step 7 of this flow |
| `Completed` | No | Step 12 of this flow |
| `FailedNeedsRecovery` | **Yes** | Step 5, 8, Catch scope |
| `MetadataSyncFailed` | **Yes** | Step 9 of this flow |
| `CounterSyncFailed` | **Yes** | Step 10 of this flow |

---

## Input parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `sviPredmetiItemId` | Integer | Yes | ID of the `Svi predmeti` item that needs a number |
| `dokumentaItemId` | Integer | Yes | ID of the `Dokumenta` file item |
| `registryBookKey` | String | Yes | Key from AppConfig `DelovodneKnjige`; used to build `LockKey` and `DelovodnaKnjigaKey` |
| `activeYear` | Integer | Yes | Active year for this registry book |
| `useReservedNumber` | Boolean | Yes | `true` if the caller is using a specific reserved number |
| `reservedNumber` | Integer | No | Required when `useReservedNumber = true`; the specific number to assign |
| `reservedNumberItemId` | Integer | No | ID of the `Rezervisani brojevi` item to delete after successful use |
| `requestedByEmail` | String | Yes | Email of the user who initiated the registration |

---

## Output schema

```json
{
  "success": true,
  "code": "OK",
  "message": "Delovodni broj DK01-6/2026 uspešno dodeljen.",
  "numberAssigned": true,
  "itemId": 101,
  "documentId": 55,
  "delovodniBrojNumber": 6,
  "delovodniBrojText": "DK01-6/2026",
  "registryYear": 2026,
  "technicalStatus": "Completed",
  "requiresRecovery": false,
  "blockNewRegistrations": false,
  "technicalDetail": ""
}
```

### Output field descriptions

| Field | Type | Notes |
|---|---|---|
| `success` | Boolean | `true` only when number is assigned, both lists updated, and no recovery required |
| `code` | String | Machine-readable result code — see table below |
| `message` | String | User-facing message in Serbian |
| `numberAssigned` | Boolean | `true` when `DelovodniBrojNumber` has been written to `Svi predmeti`, even if sync or counter update is still pending. `false` when no number was written at all. |
| `itemId` | Integer | Echo of `sviPredmetiItemId`; caller correlation |
| `documentId` | Integer | Echo of `dokumentaItemId`; caller correlation |
| `delovodniBrojNumber` | Integer | Numeric value assigned; `0` if not assigned |
| `delovodniBrojText` | String | Formatted display string; empty if not assigned |
| `registryYear` | Integer | Year for which the number was assigned |
| `technicalStatus` | String | Final `TechnicalStatus` written to `Svi predmeti` |
| `requiresRecovery` | Boolean | `true` when data is in an inconsistent state requiring admin action |
| `blockNewRegistrations` | Boolean | `true` when `CF_DocCentral_CreateDocumentTransaction` must signal Canvas to suspend the registration form |
| `technicalDetail` | String | Internal error detail; written to AuditLog; not shown to end users |

### Output codes

| Code | `success` | `numberAssigned` | `requiresRecovery` | `blockNewRegistrations` | Meaning |
|---|---|---|---|---|---|
| `OK` | **true** | true | false | false | Number assigned; both lists updated; all cleanup complete |
| `SYNC_PENDING_RECOVERY_REQUIRED` | **false** | **true** | **true** | **true** | Number written to Svi predmeti; Dokumenta sync failed; recovery required; new registrations blocked |
| `COUNTER_SYNC_FAILED` | **false** | **true** | **true** | **true** | Number written to Svi predmeti; AppConfig counter update failed; next run cannot safely determine next number without repair |
| `LOCK_BUSY` | false | false | false | false | Another flow holds the lock; no data written; caller may retry |
| `LOCK_EXPIRED` | false | false | false | **true** | Expired lock from a previous run detected; transaction state unknown; admin must validate |
| `BLOCKED_RECOVERY_REQUIRED` | false | false | false | **true** | FailedNeedsRecovery, MetadataSyncFailed, or CounterSyncFailed item exists; admin must resolve |
| `NEXT_NUMBER_IS_RESERVED` | false | false | false | **true** | Next sequential number is reserved; automatic assignment is blocked; operator must use reserved path or cancel the reservation |
| `COUNTER_OUT_OF_SYNC` | false | false | false | **true** | Svi predmeti max > AppConfig counter; counter is behind actual assignments; recovery required |
| `COUNTER_AHEAD_NEEDS_REVIEW` | false | false | false | **true** | AppConfig counter > Svi predmeti max by more than expected; possible skipped numbers; admin must verify |
| `DUPLICATE_DETECTED` | false | false | **true** | **true** | Candidate number already exists in Svi predmeti; critical data integrity issue; FailedNeedsRecovery set |
| `RESERVED_NOT_FOUND` | false | false | false | false | Reserved number not in Rezervisani brojevi; no data written |
| `ASSIGN_FAILED` | false | false | **true** | **true** | Write to Svi predmeti failed; FailedNeedsRecovery set |
| `INTERNAL_ERROR` | false | false | **true** | **true** | Unhandled exception; FailedNeedsRecovery set if data was partially written |

> **Note on `NEXT_NUMBER_IS_RESERVED`:** `blockNewRegistrations = true` because the sequence is stuck. The caller cannot safely assign the next sequential number without resolving the reserved number first. The operator must explicitly either switch to the reserved path or cancel the reservation.

> **Caller responsibility:** `CF_DocCentral_CreateDocumentTransaction` must check `blockNewRegistrations` and, if `true`, write a blocking flag that Canvas app reads on next load.

---

## Variable initialization

Declare all variables before the Try scope.

```
varLockKey                    (String)   concat(inputs.registryBookKey, '_', string(inputs.activeYear))
varLockItemId                 (Integer)  0
varLockAcquired               (Boolean)  false
varLockBelongsToThisRun       (Boolean)  false
varNextNumber                 (Integer)  0
varDelovodniBrojText          (String)   ""
varNumberAssigned             (Boolean)  false
varFlowSuccess                (Boolean)  false
varSyncSuccess                (Boolean)  false
varCounterUpdateSuccess       (Boolean)  false
varOutputCode                 (String)   "INTERNAL_ERROR"
varOutputMessage              (String)   ""
varTechnicalDetail            (String)   ""
varRequiresRecovery           (Boolean)  false
varBlockNewRegistrations      (Boolean)  false
varConfigCounter              (Integer)  0
varMaxAssigned                (Integer)  0
varAppConfigItemId            (Integer)  0
varAppConfigJson              (String)   ""
```

---

## Detailed action sequence

### TRY SCOPE

All write operations are inside Try. Catch handles unhandled exceptions. Finally handles lock cleanup. The flow always reaches the Respond action at the end.

---

**Step 1 — Attempt to create the lock**

Action: SharePoint — Create item in `NumberLocks`

```
Title:             varLockKey
LockKey:           varLockKey
LockedBy:          inputs.requestedByEmail
LockedAt:          utcNow()
ExpiresAt:         addMinutes(utcNow(), 5)
FlowRunId:         workflow()?['run']?['id']
Status:            Active
RelatedItemId:     inputs.sviPredmetiItemId
RelatedDocumentId: inputs.dokumentaItemId
```

**Retry policy: None.** A duplicate key conflict (HTTP 400) is an expected business condition, not a transient error.

After the action, branch on success vs failure:

**On success:**
```
Set varLockItemId = outputs('Create_NumberLock')?['body/ID']
Set varLockAcquired = true
Set varLockBelongsToThisRun = true
→ Continue to Step 1b
```

**On failure (any error — conflict or otherwise):**
```
Set varOutputCode = "LOCK_BUSY"
Set varOutputMessage = "Sistem je trenutno zauzet dodavanjem delovodnog broja. Pokušajte ponovo za nekoliko sekundi."
Set varTechnicalDetail = concat("Lock create failed on key: ", varLockKey,
                                " | Error: ", actions('Create_NumberLock')?['error']?['message'])
Write AuditLog: LockCollisionDetected, Warning
→ Fall through to Finally (varLockBelongsToThisRun = false; no lock to release)
```

Implementation note: wrap all subsequent steps inside a condition block gated on `varLockBelongsToThisRun eq true`. This replaces `Terminate` and ensures the Respond action always executes.

---

**Step 1b — Verify no other Active lock with same key**

Defends against the rare case where the `LockKey` unique constraint was not yet active when the lock was created.

Action: SharePoint — Get items from `NumberLocks`

```
Filter: LockKey eq 'varLockKey' and Status eq 'Active'
Top:    5
Select: ID, FlowRunId, ExpiresAt
```

Condition: `length(body('Get_active_locks')?['value']) > 1`

If true:
```
Set varOutputCode = "LOCK_BUSY"
Set varOutputMessage = "Sistem je trenutno zauzet."
Set varLockBelongsToThisRun = false
Write AuditLog: LockCollisionDetected, Warning, TechnicalDetail = "Multiple active locks detected for same key"
→ Exit condition block (fall through to Finally; lock will not be released since varLockBelongsToThisRun = false)
```

---

**Step 1c — Detect expired Active lock from a previous run**

Action: SharePoint — Get items from `NumberLocks`

```
Filter: LockKey eq 'varLockKey' and Status eq 'Active' and ExpiresAt lt 'utcNow()' and ID ne varLockItemId
Top:    1
Select: ID, ExpiresAt, RelatedItemId, FlowRunId
```

Condition: `length(body('Get_expired_locks')?['value']) > 0`

If true:
```
Set varOutputCode = "LOCK_EXPIRED"
Set varOutputMessage = "Prethodna operacija nije završena. Administrator mora proveriti stanje registracije pre nastavka."
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("Expired lock ID: ", first(body('Get_expired_locks')?['value'])?['ID'],
                                " | ExpiresAt: ", first(body('Get_expired_locks')?['value'])?['ExpiresAt'])
Write AuditLog: LockExpiredDetected, Warning
→ Exit condition block (our lock released in Finally; no data written)
```

---

**Step 2 — Check for blocking TechnicalStatus items**

Blocks if any `Svi predmeti` item is in a state that indicates unresolved data inconsistency.

Action: SharePoint — Get items from `Svi predmeti`

```
Filter: TechnicalStatus eq 'FailedNeedsRecovery'
        or TechnicalStatus eq 'MetadataSyncFailed'
        or TechnicalStatus eq 'CounterSyncFailed'
Top:    1
Select: ID, TechnicalStatus
```

Condition: `length(body('Get_blocking_items')?['value']) > 0`

If true:
```
Set varOutputCode = "BLOCKED_RECOVERY_REQUIRED"
Set varOutputMessage = "Registracija je privremeno blokirana. Administrator mora rešiti otvorene probleme pre nastavka."
Set varBlockNewRegistrations = true
Write AuditLog: NumberAssignmentBlocked, Warning,
    TechnicalDetail = concat("Blocking item ID: ", first(body('Get_blocking_items')?['value'])?['ID'],
                             " | TechnicalStatus: ", first(body('Get_blocking_items')?['value'])?['TechnicalStatus'])
→ Exit condition block (our lock released in Finally; no data written)
```

---

**Step 3 — Read AppConfig DelovodneKnjige**

Action: SharePoint — Get items from `AppConfig`

```
Filter: Title eq 'DelovodneKnjige'
Top:    1
Select: ID, Config
```

Set `varAppConfigItemId = first(body(...))?['ID']`

Parse JSON on the `Config` column value.

Find the array entry where `Key eq inputs.registryBookKey AND IsActive eq true AND ActiveYear eq inputs.activeYear`.

If no matching entry found:
```
Set varOutputCode = "INTERNAL_ERROR"
Set varOutputMessage = "Aktivna delovodna knjiga nije pronađena u konfiguraciji."
Set varBlockNewRegistrations = false
→ Exit condition block (our lock released in Finally; no data written)
```

Extract:
```
varConfigCounter = entry.CurrentCounter  (integer)
varAppConfigJson = full Config JSON string (needed for step 10 reconstruction)
```

---

**Step 4 — Determine next number and validate counter integrity**

Action: SharePoint — Get items from `Svi predmeti`

```
OData filter: DelovodnaKnjigaKey eq 'inputs.registryBookKey'
              and DelovodniBrojGodina eq inputs.activeYear
$orderby:     DelovodniBrojNumber desc
$top:         1
$select:      ID, DelovodniBrojNumber
```

Set:
```
varMaxAssigned = if item found: int(first item DelovodniBrojNumber) else: 0
varNextNumber  = varConfigCounter + 1
```

**Counter integrity checks:**

**Condition A — Svi predmeti ahead of counter (`varMaxAssigned > varConfigCounter`)**

`Svi predmeti` contains numbers higher than the AppConfig counter. The counter is behind actual assignments. This means at least one previous AppConfig update failed (REC-005) or the counter was reset incorrectly. This is not safe to ignore.

```
Set varOutputCode = "COUNTER_OUT_OF_SYNC"
Set varOutputMessage = "Brojač delovodnih brojeva nije sinhronizovan sa Svi predmeti. Administrator mora proveriti stanje pre nastavka."
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("AppConfig counter=", varConfigCounter,
                                " | Svi predmeti max=", varMaxAssigned,
                                " | Book=", inputs.registryBookKey,
                                " | Year=", inputs.activeYear)
Write AuditLog: CounterOutOfSync, Warning
→ Exit condition block (our lock released in Finally; no data written)
```

**Condition B — Counter consistent (`varMaxAssigned eq varConfigCounter`)**

System is consistent. `varNextNumber = varConfigCounter + 1`. Continue.

**Condition C — Counter ahead of Svi predmeti (`varMaxAssigned lt varConfigCounter`)**

AppConfig counter is higher than the highest number found in `Svi predmeti`. This may indicate:
- A previous transaction incremented the counter but the Svi predmeti write failed → gap in sequence → `FailedNeedsRecovery` should already be blocking (caught at step 2)
- Counter was manually set incorrectly
- Items were deleted from Svi predmeti without counter correction

This is not automatically safe. Block and require admin review unless the gap is exactly 1 (which can happen during step 10 lag in normal operation).

```
If (varConfigCounter - varMaxAssigned) > 1:
    Set varOutputCode = "COUNTER_AHEAD_NEEDS_REVIEW"
    Set varOutputMessage = "Brojač delovodnih brojeva je ispred stvarnih zapisa. Administrator mora proveriti da li postoje preskočeni brojevi."
    Set varBlockNewRegistrations = true
    Set varTechnicalDetail = concat("AppConfig counter=", varConfigCounter,
                                    " | Svi predmeti max=", varMaxAssigned,
                                    " | Gap=", sub(varConfigCounter, varMaxAssigned),
                                    " | Book=", inputs.registryBookKey,
                                    " | Year=", inputs.activeYear)
    Write AuditLog: CounterAheadNeedsReview, Warning
    → Exit condition block (our lock released in Finally; no data written)

If (varConfigCounter - varMaxAssigned) eq 1:
    — Gap of exactly 1 is within normal tolerance during high-concurrency or after
    — a counter update that preceded the Svi predmeti write in a previous run.
    — Continue with varNextNumber = varConfigCounter + 1.
    Write AuditLog: CounterGapOfOne, Info (for diagnostics)
```

**Reserved number path override**

Applied after all integrity checks, and only if `inputs.useReservedNumber eq true`:

```
Set varNextNumber = inputs.reservedNumber
— Reserved number bypasses counter calculation; do not re-run integrity checks for the reserved number itself.
— The reserved number may be lower than the counter (it was reserved in the past).
```

---

**Step 5 — Verify candidate number is not already assigned**

Action: SharePoint — Get items from `Svi predmeti`

```
Filter: DelovodniBrojNumber eq varNextNumber
        and DelovodnaKnjigaKey eq 'inputs.registryBookKey'
        and DelovodniBrojGodina eq inputs.activeYear
Top:    1
Select: ID, DelovodniBrojNumber, DelovodniBrojText
```

Condition: `length(body('Get_duplicate_check')?['value']) > 0`

If duplicate found:
```
Set varOutputCode = "DUPLICATE_DETECTED"
Set varRequiresRecovery = true
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("Duplicate number ", varNextNumber, " already exists in Svi predmeti item ID: ",
                                 first(body('Get_duplicate_check')?['value'])?['ID'])
Write AuditLog: NumberAssignmentFailed, Critical
Update Svi predmeti (inputs.sviPredmetiItemId): TechnicalStatus = FailedNeedsRecovery
Update NumberLocks (varLockItemId): Status = Failed, ErrorCode = "DUPLICATE_DETECTED"
Set varLockBelongsToThisRun = false
→ Exit condition block
```

---

**Step 6 — Verify candidate number against Rezervisani brojevi**

**Branch A: `useReservedNumber = false`**

Action: SharePoint — Get items from `Rezervisani brojevi`

```
Filter: RezervisaniBroj eq varNextNumber
Top:    1
Select: ID, RezervisaniBroj
```

Condition: `length(body('Get_reserved_check')?['value']) > 0`

If reserved entry found:
```
Set varOutputCode = "NEXT_NUMBER_IS_RESERVED"
Set varOutputMessage = concat("Sledeći redni broj (", varNextNumber, ") je rezervisan. ",
                               "Operator mora odabrati rezervisani put ili otkazati rezervaciju pre nastavka.")
Set varRequiresRecovery = false
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("Number ", varNextNumber, " found in Rezervisani brojevi, item ID: ",
                                 first(body('Get_reserved_check')?['value'])?['ID'])
Write AuditLog: NextNumberReserved, Warning
→ Exit condition block (our lock released in Finally; no data written)
```

**Do not increment `varNextNumber` and retry.** Incrementing past a reserved number constitutes skipping, which is prohibited.

**Branch B: `useReservedNumber = true`**

Action: SharePoint — Get items from `Rezervisani brojevi`

```
Filter: RezervisaniBroj eq inputs.reservedNumber
Top:    1
Select: ID, RezervisaniBroj
```

Condition: `length(body('Get_reserved_verify')?['value']) eq 0`

If not found:
```
Set varOutputCode = "RESERVED_NOT_FOUND"
Set varOutputMessage = "Rezervisani broj nije pronađen. Možda je već iskorišćen ili je broj pogrešan."
→ Exit condition block (our lock released in Finally; no data written)
```

---

**Step 7 — Set TechnicalStatus = NumberAssignmentInProgress**

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID:         inputs.sviPredmetiItemId
TechnicalStatus: NumberAssignmentInProgress
```

Retry policy: Fixed, Count = 3, Interval = PT5S.

If update fails:
```
Set varOutputCode = "ASSIGN_FAILED"
Set varRequiresRecovery = true
Set varBlockNewRegistrations = true
Write AuditLog: NumberAssignmentFailed, Critical, TechnicalDetail = error message
Update NumberLocks (varLockItemId): Status = Failed, ErrorCode = "ASSIGN_FAILED_PRESTATUS"
Set varLockBelongsToThisRun = false
→ Exit condition block
```

---

**Step 8 — Write DelovodniBroj columns to Svi predmeti**

Build the formatted text:
```
Set varDelovodniBrojText = concat(inputs.registryBookKey, '-', string(varNextNumber), '/', string(inputs.activeYear))
```

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID:               inputs.sviPredmetiItemId
DelovodniBrojNumber:   varNextNumber
DelovodniBrojText:     varDelovodniBrojText
DelovodniBrojGodina:   inputs.activeYear
DelovodnaKnjigaKey:    inputs.registryBookKey
```

Retry policy: Fixed, Count = 3, Interval = PT5S.

If update succeeds:
```
Set varNumberAssigned = true
```

If update fails:
```
Set varOutputCode = "ASSIGN_FAILED"
Set varRequiresRecovery = true
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("Step 8 Svi predmeti write failed: ", error message)
Write AuditLog: NumberAssignmentFailed, Critical
Update Svi predmeti (inputs.sviPredmetiItemId): TechnicalStatus = FailedNeedsRecovery
Update NumberLocks (varLockItemId): Status = Failed, ErrorCode = "ASSIGN_FAILED"
Set varLockBelongsToThisRun = false
→ Exit condition block
```

---

**Step 9 — Write DelovodniBroj columns to Dokumenta**

This step runs only when `varNumberAssigned = true`.

Action: SharePoint — Update file properties in `Dokumenta`

```
Item ID:               inputs.dokumentaItemId
DelovodniBrojNumber:   varNextNumber
DelovodniBrojText:     varDelovodniBrojText
DelovodniBrojGodina:   inputs.activeYear
DelovodnaKnjigaKey:    inputs.registryBookKey
```

Retry policy: Fixed, Count = 3, Interval = PT5S.

If update succeeds:
```
Set varSyncSuccess = true
```

If update fails:
```
Set varSyncSuccess = false
Set varOutputCode = "SYNC_PENDING_RECOVERY_REQUIRED"
Set varOutputMessage = "Delovodni broj je zaveden u Svi predmeti ali sinhronizacija sa bibliotekom dokumenata nije uspela. Registracija je blokirana do oporavka."
Set varRequiresRecovery = true
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat("Step 9 Dokumenta sync failed: ", error message)
Update Svi predmeti (inputs.sviPredmetiItemId): TechnicalStatus = MetadataSyncFailed
Write AuditLog: MetadataSyncFailed, Warning
— varNumberAssigned remains true (number IS in Svi predmeti)
— varLockBelongsToThisRun remains true (lock will be Released in Finally)
— continue to step 10 to update the counter (the number is assigned)
```

---

**Step 10 — Update AppConfig CurrentCounter**

This step runs only when `varNumberAssigned = true`.

Reconstruct the AppConfig `Config` JSON: replace `CurrentCounter` for the matching registry book entry with `varNextNumber`. All other entries must remain unchanged.

Action: SharePoint — Update item in `AppConfig`

```
Item ID: varAppConfigItemId
Config:  [reconstructed JSON string]
```

Retry policy: Fixed, Count = 3, Interval = PT5S.

If update succeeds:
```
Set varCounterUpdateSuccess = true
```

If update fails:
```
Set varCounterUpdateSuccess = false
Set varOutputCode = "COUNTER_SYNC_FAILED"
Set varOutputMessage = "Delovodni broj je zaveden ali ažuriranje brojača nije uspelo. Registracija je blokirana do sinhronizacije brojača."
Set varRequiresRecovery = true
Set varBlockNewRegistrations = true
Set varTechnicalDetail = concat(varTechnicalDetail,
                                " | Step 10 AppConfig counter update failed: ", error message,
                                " | AssignedNumber=", varNextNumber,
                                " | Book=", inputs.registryBookKey,
                                " | Year=", inputs.activeYear)
Update Svi predmeti (inputs.sviPredmetiItemId): TechnicalStatus = CounterSyncFailed
Write AuditLog: AppConfigUpdateFailed, Warning,
    TechnicalDetail = varTechnicalDetail
— varNumberAssigned remains true
— varLockBelongsToThisRun remains true (lock will be Released in Finally)
— do NOT continue to step 11 or 12 if counter update failed; fall through to Finally
```

The reason this blocks: if the counter is not updated, the next flow run will read the old `CurrentCounter` and calculate the same `varNextNumber` again. The step 5 duplicate check would catch it, but it would set `FailedNeedsRecovery` on the new item. It is cleaner to block here and let recovery update the counter, confirming sequence safety before any new assignment.

---

**Step 11 — Delete used reserved number (reserved path only)**

This step runs only when:
- `inputs.useReservedNumber = true`
- `varSyncSuccess = true`
- `varCounterUpdateSuccess = true`

> Only delete after both Svi predmeti and Dokumenta are confirmed written and the counter is updated. Premature deletion when sync or counter failed would leave the sequence in an ambiguous state.

Action: SharePoint — Delete item from `Rezervisani brojevi`

```
Item ID: inputs.reservedNumberItemId
```

Retry policy: Fixed, Count = 3, Interval = PT5S.

If delete fails:
```
Write AuditLog: ReservedNumberDeletionFailed, Warning,
    TechnicalDetail = concat("reservedNumberItemId=", inputs.reservedNumberItemId, " | Error=", error message)
Update NumberLocks (varLockItemId): ErrorCode = "RESERVED_DELETE_FAILED"
— do NOT set FailedNeedsRecovery; recovery flow handles REC-003
— varFlowSuccess remains on track for the success path
```

If delete succeeds:
```
Write AuditLog: ReservedNumberUsed, Info,
    TechnicalDetail = concat("reservedNumber=", inputs.reservedNumber, " | itemId=", inputs.reservedNumberItemId)
```

---

**Step 12 — Set TechnicalStatus = Completed**

Condition: `varSyncSuccess eq true AND varCounterUpdateSuccess eq true`

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID:         inputs.sviPredmetiItemId
TechnicalStatus: Completed
```

---

**Step 13 — Write AuditLog success entry**

Condition: `varSyncSuccess eq true AND varCounterUpdateSuccess eq true`

```
Write AuditLog:
    EventType:         NumberAssigned
    Severity:          Info
    Title:             concat("Delovodni broj ", varDelovodniBrojText, " dodeljen")
    RelatedItemId:     inputs.sviPredmetiItemId
    RelatedDocumentId: inputs.dokumentaItemId
    UserEmail:         inputs.requestedByEmail
    TechnicalDetails:  concat("Number=", varNextNumber,
                               " | Text=", varDelovodniBrojText,
                               " | Book=", inputs.registryBookKey,
                               " | Year=", inputs.activeYear,
                               " | Reserved=", string(inputs.useReservedNumber))
```

---

**Step 14 — Set success output variables**

Condition: `varSyncSuccess eq true AND varCounterUpdateSuccess eq true`

```
Set varFlowSuccess           = true
Set varOutputCode            = "OK"
Set varOutputMessage         = concat("Delovodni broj ", varDelovodniBrojText, " uspešno dodeljen.")
Set varRequiresRecovery      = false
Set varBlockNewRegistrations = false
```

---

### CATCH SCOPE

Runs only on an unhandled exception that escaped all inline failure handling in Try.

```
Compose technical error:
    concat("Unhandled exception in CF_DocCentral_AssignRegistryNumber",
           " | Action: ", actions()?['name'],
           " | Code: ", result()?[0]?['error']?['code'],
           " | Message: ", result()?[0]?['error']?['message'])

Set varTechnicalDetail       = composed error string
Set varOutputCode            = "INTERNAL_ERROR"
Set varOutputMessage         = "Interna greška prilikom dodele delovodnog broja. Kontaktirajte administratora."
Set varRequiresRecovery      = true
Set varBlockNewRegistrations = true

Write AuditLog:
    EventType:        NumberAssignmentFailed
    Severity:         Critical
    TechnicalDetails: varTechnicalDetail

Condition: varLockAcquired = true AND varLockBelongsToThisRun = true AND varNumberAssigned = false
    — Only set FailedNeedsRecovery if we have not yet written a number.
    — If varNumberAssigned = true, TechnicalStatus was already set to MetadataSyncFailed or CounterSyncFailed
    — by the inline handler; do not overwrite.
    Update Svi predmeti (inputs.sviPredmetiItemId): TechnicalStatus = FailedNeedsRecovery
    Update NumberLocks (varLockItemId): Status = Failed, ErrorCode = "INTERNAL_ERROR"
    Set varLockBelongsToThisRun = false
```

---

### FINALLY SCOPE

Runs unconditionally after Try (or after Catch). Manages lock state.

The key variable is `varLockBelongsToThisRun`. When a failure handler sets it to `false`, it means the lock was already updated to `Failed` by that handler. Finally must not overwrite it.

```
Condition: varLockAcquired = true AND varLockItemId > 0 AND varLockBelongsToThisRun = true

    Sub-condition: varFlowSuccess = true
        — Full success. Lock purpose fulfilled.
        Update NumberLocks (varLockItemId): Status = Released
        (Optionally delete for list hygiene.)

    Sub-condition: varFlowSuccess = false AND varNumberAssigned = true
        — Number was written to Svi predmeti. Sync or counter failed.
        — (SYNC_PENDING_RECOVERY_REQUIRED or COUNTER_SYNC_FAILED)
        — The number is real and safe; lock can be released.
        Update NumberLocks (varLockItemId): Status = Released

    Sub-condition: varFlowSuccess = false AND varNumberAssigned = false AND varRequiresRecovery = false
        — Business rejection before any write: LOCK_BUSY, BLOCKED_RECOVERY_REQUIRED,
          NEXT_NUMBER_IS_RESERVED, RESERVED_NOT_FOUND, COUNTER_OUT_OF_SYNC,
          COUNTER_AHEAD_NEEDS_REVIEW, LOCK_EXPIRED.
          Nothing was written; nothing to recover.
        Update NumberLocks (varLockItemId): Status = Released
        (Optionally delete for list hygiene.)

Condition: varLockBelongsToThisRun = false
    — Lock was set to Failed by a failure handler (ASSIGN_FAILED, DUPLICATE_DETECTED, INTERNAL_ERROR).
    — Do NOT update the lock. Leave it as Failed for the recovery audit.
    — No action.
```

**Complete lock state outcome table:**

| Scenario | `varLockBelongsToThisRun` | `varFlowSuccess` | `varNumberAssigned` | `varRequiresRecovery` | Lock final `Status` |
|---|---|---|---|---|---|
| `OK` | true | true | true | false | Released |
| `SYNC_PENDING_RECOVERY_REQUIRED` | true | false | true | true | Released |
| `COUNTER_SYNC_FAILED` | true | false | true | true | Released |
| Business rejection, no write | true | false | false | false | Released |
| `ASSIGN_FAILED` / `DUPLICATE_DETECTED` / `INTERNAL_ERROR` | false | false | false | true | Failed (set by handler) |
| `LOCK_BUSY` — lock never created | false | false | false | false | Not touched |

---

### RESPOND TO CALLER

This action is placed at the top level of the flow, after the Finally scope. It always executes regardless of the path taken.

```
Respond to a PowerApp or Flow:
    success:               varFlowSuccess
    code:                  varOutputCode
    message:               varOutputMessage
    numberAssigned:        varNumberAssigned
    itemId:                inputs.sviPredmetiItemId
    documentId:            inputs.dokumentaItemId
    delovodniBrojNumber:   varNextNumber
    delovodniBrojText:     varDelovodniBrojText
    registryYear:          inputs.activeYear
    technicalStatus:       [derive from varOutputCode or re-read from Svi predmeti]
    requiresRecovery:      varRequiresRecovery
    blockNewRegistrations: varBlockNewRegistrations
    technicalDetail:       varTechnicalDetail
```

---

## AuditLog entries — field mapping

| EventType | Severity | When triggered | TechnicalDetails must include |
|---|---|---|---|
| `NumberAssigned` | Info | Step 13 | assignedNumber, delovodniBrojText, registryBookKey, year, useReservedNumber |
| `NumberAssignmentFailed` | Critical | Steps 5, 8 failure, Catch scope | Step where failure occurred, varNextNumber, error message |
| `NumberAssignmentBlocked` | Warning | Step 2 | Blocking item ID, blocking TechnicalStatus value |
| `NextNumberReserved` | Warning | Step 6 Branch A | varNextNumber, Rezervisani brojevi item ID |
| `CounterOutOfSync` | Warning | Step 4 Condition A | varConfigCounter, varMaxAssigned, registryBookKey, year |
| `CounterAheadNeedsReview` | Warning | Step 4 Condition C (gap > 1) | varConfigCounter, varMaxAssigned, gap size, registryBookKey, year |
| `CounterGapOfOne` | Info | Step 4 Condition C (gap = 1) | varConfigCounter, varMaxAssigned, registryBookKey, year |
| `LockCollisionDetected` | Warning | Step 1 failure, Step 1b | varLockKey, competing FlowRunId if available |
| `LockExpiredDetected` | Warning | Step 1c | Expired lock ID, ExpiresAt, RelatedItemId |
| `ReservedNumberUsed` | Info | Step 11 delete success | reservedNumber, reservedNumberItemId |
| `ReservedNumberDeletionFailed` | Warning | Step 11 delete failure | reservedNumberItemId, error message |
| `MetadataSyncFailed` | Warning | Step 9 failure | dokumentaItemId, error message |
| `AppConfigUpdateFailed` | Warning | Step 10 failure | registryBookKey, varNextNumber, error message |

---

## Retry policy summary

| Action | Retry policy | Reason |
|---|---|---|
| Create lock item (Step 1) | **None** | Duplicate key conflict is expected; retrying would cause the flow to queue unpredictably |
| All SharePoint update/write actions | **Fixed, Count=3, Interval=PT5S** | Transient SharePoint throttling |
| AuditLog writes | **Fixed, Count=2, Interval=PT3S** | Best-effort; AuditLog failure must not block the main transaction path |

---

## Parallel execution note

No write actions run in parallel. All SharePoint writes follow the sequence defined above. Read-only actions at step 3 and step 4 may be placed in a Parallel branch to reduce latency, but only if both complete before any write action begins.

---

## Dependencies — must be confirmed before implementation

| Dependency | Required before |
|---|---|
| `NumberLocks` list created with `LockKey` unique + indexed | Step 1 |
| `TechnicalStatus` Choice column on `Svi predmeti` — values must include `CounterSyncFailed` | Steps 2, 7, 8, 10, 12 |
| `DelovodniBrojNumber` (Number, indexed, not unique) on `Svi predmeti` | Steps 4, 5, 8 |
| `DelovodniBrojText` (Single line text, indexed, unique) on `Svi predmeti` | Step 8 |
| `DelovodniBrojGodina` (Number, indexed) on `Svi predmeti` | Steps 4, 5, 8 |
| `DelovodnaKnjigaKey` (Single line text, indexed) on `Svi predmeti` | Steps 4, 5, 8 |
| Same four DelovodniBroj columns on `Dokumenta` | Step 9 |
| `EV_DocCentralV3_lstNumberLocks` environment variable | Step 1 |
| Internal column names confirmed and returned | All OData filters |
| AppConfig `DelovodneKnjige` JSON schema includes `Key`, `IsActive`, `ActiveYear`, `CurrentCounter` | Step 3 |

---

## What developer must return to Claude after implementation

```
Flow exact name as created:
Flow solution name:
Input parameter internal names (as configured in trigger):
  sviPredmetiItemId:
  dokumentaItemId:
  registryBookKey:
  activeYear:
  useReservedNumber:
  reservedNumber:
  reservedNumberItemId:
  requestedByEmail:
Output field internal names:
  success:
  code:
  message:
  numberAssigned:
  itemId:
  documentId:
  delovodniBrojNumber:
  delovodniBrojText:
  registryYear:
  technicalStatus:
  requiresRecovery:
  blockNewRegistrations:
  technicalDetail:
EV_DocCentralV3_lstNumberLocks variable name confirmed:
TechnicalStatus choice values as configured (exact internal values):
Internal column names — Svi predmeti:
  TechnicalStatus:
  DelovodniBrojNumber:
  DelovodniBrojText:
  DelovodniBrojGodina:
  DelovodnaKnjigaKey:
Internal column names — Dokumenta:
  DelovodniBrojNumber:
  DelovodniBrojText:
  DelovodniBrojGodina:
  DelovodnaKnjigaKey:
DelovodniBrojNumber unique constraint: confirmed NOT set (yes/no):
DelovodniBrojText unique constraint: confirmed SET (yes/no):
Confirmed tested in isolation (not via CreateDocumentTransaction): yes/no
Test results for: NL-001, NL-002, NL-010, NL-011, NL-020, NL-030, NL-040, NL-045, NL-050, NL-060, NL-065, NL-070
```

---

## Test scenarios

All tests must be run in isolation before wiring into `CF_DocCentral_CreateDocumentTransaction`.

### Normal path

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-001 | Normal assignment | Counter=5; max in Svi predmeti=5; no locks; no blocking items | `success=true`, `code=OK`, `numberAssigned=true`, `delovodniBrojNumber=6`, `delovodniBrojText="DK01-6/2026"`, `TechnicalStatus=Completed` on both lists |
| NL-002 | Reserved number assignment | `useReservedNumber=true`; `reservedNumber=3`; entry in Rezervisani brojevi | `success=true`, `code=OK`, `numberAssigned=true`, `delovodniBrojNumber=3`, reserved entry deleted |

### Counter integrity

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-040 | Svi predmeti ahead of counter | Counter=5; max in Svi predmeti=7 | `success=false`, `code=COUNTER_OUT_OF_SYNC`, `numberAssigned=false`, `blockNewRegistrations=true`, AuditLog `CounterOutOfSync` Warning |
| NL-041 | Counter ahead by gap > 1 | Counter=10; max in Svi predmeti=7 | `success=false`, `code=COUNTER_AHEAD_NEEDS_REVIEW`, `numberAssigned=false`, `blockNewRegistrations=true`, AuditLog `CounterAheadNeedsReview` Warning |
| NL-042 | Counter ahead by exactly 1 | Counter=6; max in Svi predmeti=5 | `success=true`, `code=OK`, `delovodniBrojNumber=7`, AuditLog `CounterGapOfOne` Info |
| NL-043 | Counter out of sync — after recovery | Admin corrects counter to 7; rerun | `success=true`, `code=OK`, `delovodniBrojNumber=8` |

### Next number is reserved

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-050 | Next number reserved, `useReservedNumber=false` | Counter=5; number 6 in Rezervisani brojevi | `success=false`, `code=NEXT_NUMBER_IS_RESERVED`, `numberAssigned=false`, `blockNewRegistrations=true`, AuditLog `NextNumberReserved` Warning |
| NL-051 | Operator switches to reserved path | Same setup; rerun with `useReservedNumber=true`, `reservedNumber=6` | `success=true`, `code=OK`, `delovodniBrojNumber=6`, reserved entry deleted |
| NL-052 | Operator cancels reservation; retries | Reservation deleted; rerun with `useReservedNumber=false` | `success=true`, `code=OK`, `delovodniBrojNumber=6` |

### Metadata sync failure

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-060 | Dokumenta sync fails | Simulate step 9 failure (invalid `dokumentaItemId`) | `success=false`, `code=SYNC_PENDING_RECOVERY_REQUIRED`, `numberAssigned=true`, `requiresRecovery=true`, `blockNewRegistrations=true`, `TechnicalStatus=MetadataSyncFailed` on Svi predmeti |
| NL-061 | New assignment while MetadataSyncFailed exists | Item from NL-060 still in MetadataSyncFailed | `success=false`, `code=BLOCKED_RECOVERY_REQUIRED`, `numberAssigned=false` |
| NL-062 | Admin fixes MetadataSyncFailed via recovery flow | Recovery sets `TechnicalStatus=Completed` | Next attempt: `success=true`, `code=OK` |

### AppConfig counter sync failure

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-065 | AppConfig counter update fails | Steps 8 and 9 succeed; simulate step 10 failure | `success=false`, `code=COUNTER_SYNC_FAILED`, `numberAssigned=true`, `requiresRecovery=true`, `blockNewRegistrations=true`, `TechnicalStatus=CounterSyncFailed` on Svi predmeti, AuditLog `AppConfigUpdateFailed` Warning |
| NL-066 | New assignment while CounterSyncFailed exists | Item from NL-065 still in CounterSyncFailed | `success=false`, `code=BLOCKED_RECOVERY_REQUIRED`, `numberAssigned=false` |
| NL-067 | Admin repairs counter; retries | Recovery corrects AppConfig counter; sets `TechnicalStatus=Completed` | Next attempt: `success=true`, `code=OK` |

### Locking

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-010 | Lock collision | Manually create Active lock with future ExpiresAt for same LockKey | `success=false`, `code=LOCK_BUSY`, `numberAssigned=false`, `blockNewRegistrations=false`, AuditLog `LockCollisionDetected` |
| NL-011 | 10 concurrent runs, same book | Trigger 10 instances simultaneously | Exactly 10 unique sequential numbers; concurrent runs return `LOCK_BUSY`; caller retries until all succeed |
| NL-012 | Concurrent runs, different books | 5 on DK01 and 5 on DK02 simultaneously | Each book gets 5 sequential unique numbers; no cross-book blocking |
| NL-020 | Expired lock from previous run | Manually create Active lock with ExpiresAt in the past; different `FlowRunId` | `success=false`, `code=LOCK_EXPIRED`, `numberAssigned=false`, `blockNewRegistrations=true`, AuditLog `LockExpiredDetected` |

### Blocking

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-030 | FailedNeedsRecovery blocks | Set `TechnicalStatus=FailedNeedsRecovery` on any Svi predmeti item | `success=false`, `code=BLOCKED_RECOVERY_REQUIRED`, `blockNewRegistrations=true` |
| NL-031 | MetadataSyncFailed blocks | Set `TechnicalStatus=MetadataSyncFailed` on any Svi predmeti item | `success=false`, `code=BLOCKED_RECOVERY_REQUIRED`, `blockNewRegistrations=true` |
| NL-032 | CounterSyncFailed blocks | Set `TechnicalStatus=CounterSyncFailed` on any Svi predmeti item | `success=false`, `code=BLOCKED_RECOVERY_REQUIRED`, `blockNewRegistrations=true` |
| NL-033 | All blocking states resolved | Admin sets all to Completed | Next run: `success=true`, `code=OK` |

### Duplicate and uniqueness safeguards

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-070 | Duplicate at flow level | Manually insert Svi predmeti item with same `DelovodniBrojNumber`, `DelovodnaKnjigaKey`, `DelovodniBrojGodina` | `success=false`, `code=DUPLICATE_DETECTED`, `numberAssigned=false`, `requiresRecovery=true`, `TechnicalStatus=FailedNeedsRecovery` set, AuditLog Critical |
| NL-071 | Reserved not found | `useReservedNumber=true`, `reservedNumber=99`; no entry in list | `success=false`, `code=RESERVED_NOT_FOUND`, `numberAssigned=false`, no data written |
| NL-072 | SharePoint unique constraint fires at write | `DelovodniBrojText` unique constraint rejects the write | Step 8 failure handler: `code=ASSIGN_FAILED`, `TechnicalStatus=FailedNeedsRecovery` set |

### Output schema and control flow validation

| ID | Scenario | Expected behavior |
|---|---|---|
| NL-080 | `code=OK` output inspection | `success=true`, `numberAssigned=true`, `requiresRecovery=false`, `blockNewRegistrations=false`, all ID and number fields populated |
| NL-081 | `code=SYNC_PENDING_RECOVERY_REQUIRED` output | `success=false`, `numberAssigned=true`, `requiresRecovery=true`, `blockNewRegistrations=true`, `delovodniBrojNumber > 0` |
| NL-082 | `code=COUNTER_SYNC_FAILED` output | `success=false`, `numberAssigned=true`, `requiresRecovery=true`, `blockNewRegistrations=true` |
| NL-083 | Business rejection — no Terminate used | For any `success=false` code: caller receives fully populated response object; flow does not throw an unhandled error |
| NL-084 | Catch scope fires | `code=INTERNAL_ERROR`, `success=false`, `numberAssigned=false` (unless step 8 already succeeded); no empty or null response fields |
| NL-085 | `DelovodniBrojNumber` non-unique — confirm column setting | Attempt to insert two items with same `DelovodniBrojNumber` but different years → both succeed (column is not unique) |
| NL-086 | `DelovodniBrojText` unique — confirm column setting | Attempt to insert two items with same `DelovodniBrojText` → second write rejected by SharePoint with 400 conflict |
