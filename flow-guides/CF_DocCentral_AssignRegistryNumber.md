# Flow Guide: CF_DocCentral_AssignRegistryNumber

Last updated: 2026-05-06

---

## Purpose

Assigns the next sequential `DelovodniBroj` (registry number) to a document that has already been created as a `Svi predmeti` item and a `Dokumenta` file. Uses `NumberLocks` to prevent concurrent assignment. Enforces no-duplicate and no-skip rules. Blocks if any `FailedNeedsRecovery` item exists. Supports a reserved number path.

This flow is called by `CF_DocCentral_CreateDocumentTransaction` as an internal step. It is not called directly from Canvas app.

---

## Trigger

**Type:** Child flow (called from `CF_DocCentral_CreateDocumentTransaction` using the Run a Child Flow action)

If a standalone trigger is needed for testing, use **Power Apps V2**.

---

## Connections

All SharePoint actions in this flow must use the service account connection reference `CR_DocCentralV3_SharePoint`. No user-delegated connection.

---

## Environment variables used

| Variable | Purpose |
|---|---|
| `EV_DocCentralV3_lstSviPredmeti` | URL of `Svi predmeti` SharePoint list |
| `EV_DocCentralV3_lstAppConfig` | URL of `AppConfig` SharePoint list |
| `EV_DocCentralV3_docDokumenti` | URL of `Dokumenta` SharePoint library |
| `EV_DocCentralV3_SharePointSite` | Root SharePoint site URL |

The `NumberLocks` list URL must also be stored in an environment variable. This variable does not yet exist in the catalog. **Developer must create it and return the variable name to Claude.**

---

## Input parameters

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `sviPredmetiItemId` | Integer | Yes | ID of the `Svi predmeti` item that needs a number |
| `dokumentaItemId` | Integer | Yes | ID of the `Dokumenta` file item |
| `registryBookKey` | String | Yes | Key from AppConfig DelovodneKnjige, used to build LockKey |
| `activeYear` | Integer | Yes | Active year for this registry book |
| `useReservedNumber` | Boolean | Yes | `true` if the caller is using a specific reserved number |
| `reservedNumber` | Integer | No | Required when `useReservedNumber = true`; the reserved number to use |
| `reservedNumberItemId` | Integer | No | ID of the item in `Rezervisani brojevi` to delete after use |
| `requestedByEmail` | String | Yes | Email of the user who initiated the registration |

---

## Output schema

```json
{
  "success": true,
  "code": "OK",
  "message": "Registry number assigned successfully.",
  "assignedNumber": 42,
  "technicalDetail": ""
}
```

### Output codes

| Code | Success | Meaning |
|---|---|---|
| `OK` | true | Number assigned successfully |
| `LOCK_BUSY` | false | Another flow currently holds the lock; caller should retry after a brief wait |
| `LOCK_EXPIRED` | false | An expired lock exists; recovery validation required before proceeding |
| `BLOCKED_RECOVERY_REQUIRED` | false | One or more FailedNeedsRecovery items exist; admin must resolve |
| `DUPLICATE_DETECTED` | false | The candidate number was already found in Svi predmeti; critical failure |
| `RESERVED_NOT_FOUND` | false | Reserved number not found in Rezervisani brojevi |
| `ASSIGN_FAILED` | false | Write to Svi predmeti failed; TechnicalStatus set to FailedNeedsRecovery |
| `SYNC_FAILED` | false | Write to Dokumenta failed; TechnicalStatus set to MetadataSyncFailed |
| `INTERNAL_ERROR` | false | Unhandled error in flow |

---

## Detailed action sequence

The flow is structured in three scopes: **Try**, **Catch**, and **Finally**. All SharePoint write actions are in Try. Catch handles failures. Finally releases the lock unconditionally.

### 0. Initialize variables

```
Initialize variable: varLockItemId        (Integer, 0)
Initialize variable: varLockAcquired      (Boolean, false)
Initialize variable: varAssignedNumber    (Integer, 0)
Initialize variable: varFlowSuccess       (Boolean, false)
Initialize variable: varOutputCode        (String, "INTERNAL_ERROR")
Initialize variable: varOutputMessage     (String, "")
Initialize variable: varTechnicalDetail   (String, "")
Initialize variable: varLockKey           (String, concat(triggerBody()?['registryBookKey'], '_', triggerBody()?['activeYear']))
```

---

### TRY SCOPE

**Step 1 — Attempt to create the lock**

Action: SharePoint — Create item in `NumberLocks`

```
Title:               variables('varLockKey')
LockKey:             variables('varLockKey')
LockedBy:            triggerBody()?['requestedByEmail']
LockedAt:            utcNow()
ExpiresAt:           addMinutes(utcNow(), 5)
FlowRunId:           workflow()?['run']?['id']
Status:              Active
RelatedItemId:       triggerBody()?['sviPredmetiItemId']
RelatedDocumentId:   triggerBody()?['dokumentaItemId']
```

Configure action: **Configure run after → Run this action even if the previous action failed** is NOT needed here — this is the first action.

If this action fails with HTTP 400 / "Column 'LockKey' has a duplicate value" → the lock is already held by another flow. Go to the collision handler.

Add a **Configure run after** parallel branch:

- If Create item **succeeded** → continue to Step 2
- If Create item **failed** → go to Collision Handler

**Collision Handler (parallel branch on failure):**

```
Set variable: varOutputCode = "LOCK_BUSY"
Set variable: varOutputMessage = "Sistem je trenutno zauzet dodavanjem delovodnog broja. Pokušajte ponovo za nekoliko sekundi."
Set variable: varTechnicalDetail = concat("Lock collision on key: ", variables('varLockKey'))
Terminate flow (Cancel) — this exits the Try scope and goes to Finally
```

> Note: Terminate with "Cancelled" status, not "Failed", so that Finally still runs.

After successful create:

```
Set variable: varLockItemId = outputs('Create_lock_item')?['body/ID']
Set variable: varLockAcquired = true
```

---

**Step 1b — Verify no other active lock with same key**

Even though we just created a lock, verify only one Active lock exists for this key. This catches edge cases where the unique constraint is temporarily not enforced.

Action: SharePoint — Get items from `NumberLocks`

```
Filter: LockKey eq '{varLockKey}' and Status eq 'Active'
Top: 5
```

If item count > 1:

```
Set variable: varOutputCode = "LOCK_BUSY"
Set variable: varOutputMessage = "Sistem je trenutno zauzet."
Terminate (Cancelled)
```

---

**Step 1c — Check for expired active locks from previous runs**

Action: SharePoint — Get items from `NumberLocks`

```
Filter: LockKey eq '{varLockKey}' and Status eq 'Active' and ExpiresAt lt '{utcNow()}'
Top: 1
Select: ID, ExpiresAt, RelatedItemId, FlowRunId
```

If count > 0 AND the item found is NOT the lock we just created (check by ID):

```
Write AuditLog (see AuditLog section): EventType = LockExpiredDetected
Set variable: varOutputCode = "LOCK_EXPIRED"
Set variable: varOutputMessage = "Prethodna operacija nije završena. Administrator mora proveriti stanje registracije pre nastavka."
Terminate (Cancelled)
```

---

**Step 2 — Check FailedNeedsRecovery**

Action: SharePoint — Get items from `Svi predmeti`

```
Filter: TechnicalStatus eq 'FailedNeedsRecovery'
Top: 1
Select: ID, TechnicalStatus
```

If count > 0:

```
Write AuditLog: EventType = NumberAssignmentBlocked
Set variable: varOutputCode = "BLOCKED_RECOVERY_REQUIRED"
Set variable: varOutputMessage = "Registracija je privremeno blokirana. Administrator mora rešiti otvorene probleme."
Terminate (Cancelled)
```

---

**Step 3 — Read AppConfig DelovodneKnjige**

Action: SharePoint — Get items from `AppConfig`

```
Filter: Title eq 'DelovodneKnjige'
Top: 1
Select: ID, Config
```

Parse the `Config` JSON column.

Find the active registry book entry where `Key eq registryBookKey AND IsActive eq true`.

Extract:
- `varActiveYear` — from entry's `ActiveYear` field
- `varConfigCounter` — from entry's `CurrentCounter` field
- `varAppConfigItemId` — SharePoint ID of the AppConfig item (needed for update in step 10)

If no active registry book found: set error, terminate.

---

**Step 4 — Determine next number**

Action: SharePoint — Get items from `Svi predmeti`

```
OData query:
$filter=DelovodnaKnjiga eq '{registryBookKey}' and DatumZavodjenja ge '2026-01-01' and DatumZavodjenja lt '2027-01-01'
$orderby=DelovodniBroj desc
$top=1
$select=ID,DelovodniBroj
```

> **Note:** The year filter above is illustrative. Use the actual year range calculated from `activeYear`. Replace hardcoded dates with expressions.

```
varMaxAssigned = if item found: int(first item DelovodniBroj) else: 0
varNextNumber  = max(varMaxAssigned, varConfigCounter) + 1
```

If `varConfigCounter < varMaxAssigned`: log AuditLog warning (counter was stale).

If reserved number path (`useReservedNumber = true`): `varNextNumber = reservedNumber` (the input parameter, not calculated).

---

**Step 5 — Verify candidate number is not already assigned**

Action: SharePoint — Get items from `Svi predmeti`

```
Filter: DelovodniBroj eq '{varNextNumber}'
Top: 1
```

If count > 0:

```
Set variable: varOutputCode = "DUPLICATE_DETECTED"
Set variable: varTechnicalDetail = concat("Duplicate check failed for number: ", varNextNumber)
Write AuditLog: EventType = NumberAssignmentFailed, Severity = Critical
Update Svi predmeti item TechnicalStatus = FailedNeedsRecovery
Update NumberLocks item Status = Failed, ErrorCode = DUPLICATE_DETECTED
Terminate (Cancelled)
```

---

**Step 6 — Verify candidate number against Rezervisani brojevi**

If `useReservedNumber = false`:

Action: SharePoint — Get items from `Rezervisani brojevi`

```
Filter: RezervisaniBroj eq '{varNextNumber}'
Top: 1
```

If reserved entry found: the number is reserved but caller is not on the reserved path.

```
Set variable: varOutputCode = "INTERNAL_ERROR"
Set variable: varTechnicalDetail = concat("Candidate number ", varNextNumber, " is in Rezervisani brojevi but useReservedNumber=false")
Write AuditLog: EventType = NumberAssignmentFailed, Severity = Warning
```

Recommended behavior: increment `varNextNumber` by 1 and repeat steps 5 and 6. Limit to 10 attempts to prevent infinite loop. If after 10 attempts all candidates are reserved, return `INTERNAL_ERROR`.

If `useReservedNumber = true`:

Action: SharePoint — Get items from `Rezervisani brojevi`

```
Filter: RezervisaniBroj eq '{reservedNumber}'
Top: 1
```

If not found:

```
Set variable: varOutputCode = "RESERVED_NOT_FOUND"
Set variable: varOutputMessage = "Rezervisani broj nije pronađen. Moguće je da je već iskorišćen."
Terminate (Cancelled)
```

---

**Step 7 — Set TechnicalStatus = NumberAssignmentInProgress**

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID: sviPredmetiItemId
TechnicalStatus: NumberAssignmentInProgress
```

---

**Step 8 — Write DelovodniBroj to Svi predmeti**

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID: sviPredmetiItemId
DelovodniBroj: varNextNumber
```

Configure: if this action fails, go to failure handler:

```
Update Svi predmeti TechnicalStatus = FailedNeedsRecovery
Update NumberLocks Status = Failed, ErrorCode = ASSIGN_FAILED
Write AuditLog: NumberAssignmentFailed, Severity = Critical
Set varOutputCode = ASSIGN_FAILED
Set varFlowSuccess = false
Terminate (Cancelled)
```

---

**Step 9 — Write DelovodniBroj to Dokumenta**

Action: SharePoint — Update file properties in `Dokumenta`

```
Item ID: dokumentaItemId
DelovodniBroj: varNextNumber
TechnicalStatus: Completed
```

If this action fails:

```
Update Svi predmeti TechnicalStatus = MetadataSyncFailed
Write AuditLog: MetadataSyncFailed, Severity = Warning
Set varOutputCode = SYNC_FAILED
Set varOutputMessage = "Broj je zaveden ali sinhronizacija sa bibliotekom dokumenata nije uspela. Koristite oporavak."
Set varFlowSuccess = false
— do NOT set FailedNeedsRecovery (this is lower severity)
— do NOT Terminate; continue to step 10 to update counter
```

---

**Step 10 — Update AppConfig CurrentCounter**

Action: SharePoint — Update item in `AppConfig`

Reconstruct the `Config` JSON with the updated `CurrentCounter = varNextNumber` for the matching registry book entry.

If this action fails:

```
Write AuditLog: EventType = AppConfigUpdateFailed, Severity = Warning
— do NOT set FailedNeedsRecovery
— continue
```

---

**Step 11 — Delete used reserved number (if applicable)**

Condition: `useReservedNumber = true`

Action: SharePoint — Delete item from `Rezervisani brojevi`

```
Item ID: reservedNumberItemId
```

If delete fails:

```
Write AuditLog: ReservedNumberDeletionFailed, Severity = Warning
Set ErrorCode on NumberLocks item = RESERVED_DELETE_FAILED
— do NOT set FailedNeedsRecovery
— recovery flow will retry deletion
```

---

**Step 12 — Set TechnicalStatus = Completed**

Condition: run only if `varFlowSuccess` is still true (no sync failure in step 9 set it to false) OR if you want to always mark Completed even after MetadataSyncFailed. Recommended: set Completed only if step 9 succeeded.

Action: SharePoint — Update item in `Svi predmeti`

```
Item ID: sviPredmetiItemId
TechnicalStatus: Completed
```

---

**Step 13 — Write AuditLog success**

See AuditLog entries section.

```
EventType: NumberAssigned
Severity: Info
RelatedItemId: sviPredmetiItemId
RelatedDocumentId: dokumentaItemId
UserEmail: requestedByEmail
```

Include `assignedNumber` in the TechnicalDetails field.

---

**Step 14 — Set success outputs**

```
Set variable: varAssignedNumber = varNextNumber
Set variable: varFlowSuccess = true
Set variable: varOutputCode = OK
Set variable: varOutputMessage = "Delovodni broj uspešno dodeljen."
```

---

### CATCH SCOPE

The Catch scope runs if any action inside Try throws an unhandled exception (not covered by inline failure handling above).

```
Compose technical error: concat(actions('failing_action')['error']['code'], ' — ', actions('failing_action')['error']['message'])
Set variable: varTechnicalDetail = [composed error]
Set variable: varOutputCode = INTERNAL_ERROR
Set variable: varOutputMessage = "Interna greška. Kontaktirajte administratora."

Write AuditLog:
    EventType: NumberAssignmentFailed
    Severity: Critical
    TechnicalDetails: varTechnicalDetail

If varLockAcquired = true AND Svi predmeti item TechnicalStatus was changed:
    Update Svi predmeti TechnicalStatus = FailedNeedsRecovery
    Update NumberLocks Status = Failed, ErrorCode = INTERNAL_ERROR
```

---

### FINALLY SCOPE

The Finally scope runs unconditionally after Try or Catch.

```
Condition: varLockAcquired = true AND varLockItemId > 0

If varFlowSuccess = true:
    Update NumberLocks item Status = Released
    (Optionally delete the item for housekeeping)

If varFlowSuccess = false AND varOutputCode is not LOCK_BUSY:
    Update NumberLocks item Status = Failed
    (Item remains for recovery audit)

If varOutputCode = LOCK_BUSY:
    Do NOT update the lock item (it belongs to another flow run)
```

---

### RESPOND TO CALLER

```
Respond to a PowerApp or Flow:
    success:         varFlowSuccess
    code:            varOutputCode
    message:         varOutputMessage
    assignedNumber:  varAssignedNumber
    technicalDetail: varTechnicalDetail
```

---

## AuditLog entries — field mapping

All AuditLog writes use the SharePoint list `AuditLog`. Write via the service account connection.

| EventType | Severity | Title | RelatedItemId | UserMessage | TechnicalDetails |
|---|---|---|---|---|---|
| `NumberAssigned` | Info | `"Delovodni broj {N} dodeljen"` | sviPredmetiItemId | Success message | Includes assignedNumber, registryBookKey, year |
| `NumberAssignmentFailed` | Critical | `"Dodela broja nije uspela"` | sviPredmetiItemId | User-friendly block message | Full error, step that failed, varNextNumber |
| `NumberAssignmentBlocked` | Warning | `"Dodela blokirana - oporavak potreban"` | — | Blocked message for display | FailedNeedsRecovery item IDs found |
| `LockCollisionDetected` | Warning | `"Lock kolizija - {LockKey}"` | sviPredmetiItemId | Retry message | varLockKey, competing FlowRunId if available |
| `LockExpiredDetected` | Warning | `"Istekli lock pronađen - {LockKey}"` | sviPredmetiItemId | Recovery required message | LockKey, ExpiresAt, RelatedItemId of expired lock |
| `ReservedNumberUsed` | Info | `"Iskorišćen rezervisani broj {N}"` | sviPredmetiItemId | — | reservedNumber, reservedNumberItemId |
| `ReservedNumberDeletionFailed` | Warning | `"Brisanje rezervisanog broja {N} nije uspelo"` | sviPredmetiItemId | Recovery message | reservedNumberItemId, error detail |
| `MetadataSyncFailed` | Warning | `"Sinhronizacija sa Dokumenta nije uspela"` | sviPredmetiItemId | Recovery message | dokumentaItemId, error detail |
| `AppConfigUpdateFailed` | Warning | `"AppConfig counter nije ažuriran"` | — | — | registryBookKey, varNextNumber, error detail |

---

## Retry policy

- All SharePoint create/update actions: **Retry policy = Fixed interval, Count = 3, Interval = PT5S**
- Do not retry the lock creation on duplicate key conflict — that is expected behavior, not a transient error. Configure: **Retry policy = None** on the lock Create item action, handle conflict in the failure branch.

---

## Timeout policy

- Flow timeout: default (90 days for child flows, but this flow should complete in under 30 seconds)
- Lock duration is 5 minutes, which is the practical timeout for the operation

---

## Parallel execution note

This flow does not use parallel branches for write operations. All SharePoint write actions run sequentially to ensure the transaction order is preserved. Concurrent reads (e.g., reading AppConfig and Svi predmeti max in parallel) may be done using Parallel branch but only for read-only actions before the lock write is complete.

---

## Dependencies — must be confirmed before implementation

| Dependency | Status |
|---|---|
| `NumberLocks` list created with all columns | Confirm with developer |
| `LockKey` unique constraint on `NumberLocks` | Confirm with developer |
| `TechnicalStatus` column on `Svi predmeti` | Confirm with developer |
| `DelovodniBroj` unique + indexed on `Svi predmeti` | Confirm with developer |
| `DelovodniBroj` column on `Dokumenta` | Confirm with developer |
| All environment variables pointing to correct lists | Confirm with developer |
| Service account connection reference confirmed | Confirm with developer |
| Internal column names for all lists returned to Claude | Required before implementing OData filters |

---

## What developer must return to Claude after implementation

```
Flow exact name as created:
Flow trigger type:
sviPredmetiItemId parameter internal name:
dokumentaItemId parameter internal name:
registryBookKey parameter internal name:
activeYear parameter internal name:
useReservedNumber parameter internal name:
reservedNumber parameter internal name:
reservedNumberItemId parameter internal name:
requestedByEmail parameter internal name:
Output: success field name:
Output: code field name:
Output: message field name:
Output: assignedNumber field name:
Output: technicalDetail field name:
Confirmed tested in isolation (not via CreateDocumentTransaction): yes/no
Test results for NL-001, NL-010, NL-030:
```

---

## Test scenarios

This flow must be tested independently before wiring into `CF_DocCentral_CreateDocumentTransaction`.

| ID | Scenario | Setup | Expected output |
|---|---|---|---|
| NL-001 | Normal assignment | Clean state, no locks, no FailedNeedsRecovery | code=OK, number assigned, TechnicalStatus=Completed |
| NL-002 | Reserved number | Reserved entry in Rezervisani brojevi, useReservedNumber=true | code=OK, reserved number used, entry deleted |
| NL-010 | Lock collision | Manually create Active lock for same LockKey before running flow | code=LOCK_BUSY, no number assigned, no FailedNeedsRecovery |
| NL-011 | 10 concurrent runs | Trigger 10 instances simultaneously | 10 sequential unique numbers; 9 LOCK_BUSY responses retried until success or Canvas handled |
| NL-020 | Expired lock | Manually create Active lock with ExpiresAt in the past | code=LOCK_EXPIRED, no number assigned |
| NL-030 | FailedNeedsRecovery blocks | Set TechnicalStatus=FailedNeedsRecovery on one Svi predmeti item | code=BLOCKED_RECOVERY_REQUIRED |
| NL-050 | Duplicate detection | Manually insert a Svi predmeti item with DelovodniBroj=N; run flow expecting N | code=DUPLICATE_DETECTED, FailedNeedsRecovery set, AuditLog Critical |
| NL-051 | Reserved not found | useReservedNumber=true, reservedNumber=99, no entry in Rezervisani brojevi | code=RESERVED_NOT_FOUND |
| NL-052 | SharePoint unique constraint fires | Set up scenario where duplicate write reaches SharePoint | Flow Catch handles 409, FailedNeedsRecovery set |
