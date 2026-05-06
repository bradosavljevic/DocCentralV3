# SharePoint Guide: Rezervisani brojevi (Reserved Numbers)

Last updated: 2026-05-06

---

## Purpose

`Rezervisani brojevi` is the registry of pre-reserved registry numbers. A reserved number is the only allowed exception to normal sequential assignment. When a number is reserved, it must be used through the explicit reserved-number path in `CF_DocCentral_AssignRegistryNumber` — it cannot be claimed by sequential assignment.

The list serves three roles:

1. **Reservation gate** — step 6 Branch A of `CF_DocCentral_AssignRegistryNumber` checks this list before assigning a sequential number. If the candidate next number appears here, sequential assignment is blocked. The flow returns `NEXT_NUMBER_IS_RESERVED`.

2. **Reserved-path entry** — step 6 Branch B verifies the specific reserved number exists and has not already been consumed (`UsageStatus ne 'Used'`) before the flow proceeds on the reserved path.

3. **Orphan detection** — if `UsageStatus = Used` but the item was never deleted (delete failed in step 11b), the item remains visible in the list as a cleanup obligation. It does not block any further assignment because both sequential and reserved paths detect the `Used` status.

---

## List name

Use exactly one of the following. Confirm which name is used and return to Claude before flow guides reference it.

- `Rezervisani brojevi` (Serbian — matches existing naming convention in this project)
- `ReservedNumbers` (English alternative)

---

## Required columns

| Display name | Suggested internal name | Type | Required | Notes |
|---|---|---|---|---|
| Title | Title | Single line text | Yes | Set equal to `RezervisaniBroj` formatted as text (e.g. `"DK01-15/2026"`) for readability |
| RezervisaniBroj | RezervisaniBroj | Number (integer) | Yes | **Indexed** — the raw number being reserved; see constraints section |
| DelovodnaKnjigaKey | DelovodnaKnjigaKey | Single line text | Yes | **Indexed** — registry book key, e.g. `DK01` |
| Godina | Godina | Number (integer) | Yes | **Indexed** — the year the reservation is valid for |
| RezervisaoEmail | RezervisaoEmail | Single line text | No | Email of the admin who created the reservation |
| RezervisanoVreme | RezervisanoVreme | Date and Time | Yes | UTC timestamp of reservation creation |
| Napomena | Napomena | Multiple lines of text | No | Admin note explaining why this number was reserved |
| UsageStatus | UsageStatus | Choice | Yes | See status values below; default = `Pending` |
| UsedByItemId | UsedByItemId | Number | No | SharePoint ID of the `Svi predmeti` item that consumed this number; written before delete in step 11a |
| UsedAt | UsedAt | Date and Time | No | UTC timestamp when the number was marked as used (step 11a); written before delete |

---

## UsageStatus choice values

| Value | Meaning |
|---|---|
| `Pending` | Number is reserved and available for use |
| `Used` | Number was consumed by a `Svi predmeti` item; deletion is pending or in progress |

`Used` is the reuse-prevention guard. Once set:
- Step 6 Branch A: sequential path does not block on a `Used` entry (it only blocks on `Pending`)
- Step 6 Branch B: reserved path returns `RESERVED_ALREADY_USED` and does not proceed
- The item is safe to clean up at any time via the recovery flow or a scheduled cleanup

---

## Required constraints

### RezervisaniBroj — Index

1. In SharePoint list settings → Indexed columns
2. Add index on `RezervisaniBroj`
3. Required for OData filter performance: `RezervisaniBroj eq {number}`

**Do not** set Enforce unique values on `RezervisaniBroj` alone. The same number may exist for different years or different registry books. Uniqueness is logical: `(RezervisaniBroj, DelovodnaKnjigaKey, Godina)` must not duplicate.

### Composite uniqueness enforcement

SharePoint does not support composite unique constraints. Uniqueness of `(RezervisaniBroj, DelovodnaKnjigaKey, Godina)` must be enforced by:
- The admin interface that creates reservations (Canvas app reservation form)
- The flow that creates the reservation item, which must query for an existing entry before creating
- Optionally: a dedicated validation flow called before creation

### Recommended additional indexes

| Column | Reason |
|---|---|
| `DelovodnaKnjigaKey` | Filters by registry book in reservation queries |
| `Godina` | Year-closing flow queries all reservations for the active year |
| `UsageStatus` | Cleanup queries filter by `UsageStatus eq 'Pending'` before year closing |

---

## Data lifecycle

```
[Admin creates reservation]
        │
        ▼
  UsageStatus = Pending

        │
        ▼
[CF_DocCentral_AssignRegistryNumber — step 11a]
  UsageStatus = Used
  UsedByItemId = <Svi predmeti item ID>
  UsedAt = utcNow()

        │
        ▼
[CF_DocCentral_AssignRegistryNumber — step 11b]
  Delete item from list

        │
   ┌────┴─────────────────────────────┐
   │ Delete succeeded                 │ Delete failed
   ▼                                  ▼
Item removed                   Item remains with UsageStatus=Used
(normal outcome)                (cleanup obligation; no data integrity risk)
                                Recovery flow or cleanup job deletes it
```

---

## Business rules

| Rule | Detail |
|---|---|
| Scope | Valid only for the active year (`Godina = ActiveYear` in AppConfig `DelovodneKnjige`) |
| Document type | May be used for any `TipDokumenta` |
| Year closing | All reservations with `UsageStatus = Pending` must be used or explicitly cancelled before year closing is allowed |
| Manual cancellation | A `Pending` reservation may be cancelled by an admin. The admin must write an AuditLog entry (`ReservationCancelled`) before deleting the item. |
| Manual deletion of `Pending` | Not allowed without explicit admin cancellation with AuditLog entry |
| Manual deletion of `Used` | Allowed at any time — item is already consumed; deletion is cleanup only |
| Sequential assignment blocked by `Pending` | Yes — `CF_DocCentral_AssignRegistryNumber` step 6 Branch A blocks when the candidate next number is `Pending` in this list |
| Sequential assignment blocked by `Used` | No — `Used` items are effectively invisible to the sequential path |

---

## Integration with CF_DocCentral_AssignRegistryNumber

### Step 6 Branch A — sequential path guard

The flow queries `Rezervisani brojevi` for an item where:

```
RezervisaniBroj eq varNextNumber
AND DelovodnaKnjigaKey eq inputs.registryBookKey
AND Godina eq inputs.activeYear
AND UsageStatus eq 'Pending'
```

If any item is found: return `NEXT_NUMBER_IS_RESERVED`, `blockNewRegistrations = true`. The sequential path cannot continue.

**Note:** The query must filter on `UsageStatus = 'Pending'` so that stale `Used` items (delete failed) do not block sequential assignment indefinitely.

### Step 6 Branch B — reserved path guard

The flow queries `Rezervisani brojevi` for an item where:

```
RezervisaniBroj eq inputs.reservedNumber
AND DelovodnaKnjigaKey eq inputs.registryBookKey
AND Godina eq inputs.activeYear
```

- If not found: return `RESERVED_NOT_FOUND`
- If found AND `UsageStatus eq 'Used'`: return `RESERVED_ALREADY_USED`
- If found AND `UsageStatus eq 'Pending'`: proceed with reserved path

### Step 11a — mark before delete

Before deleting the item, the flow updates it:

```
UsageStatus:   Used
UsedByItemId:  inputs.sviPredmetiItemId
UsedAt:        utcNow()
```

If this update fails: `TechnicalStatus = CounterSyncFailed` on `Svi predmeti`, `blockNewRegistrations = true`. Recovery flow must retry the mark.

### Step 11b — delete

Delete the `Rezervisani brojevi` item by ID. If this fails after step 11a succeeded: AuditLog warning only. Registrations are not blocked. The item remains as a cleanup obligation.

---

## Year closing check

Before year closing is allowed, the following query must return zero results:

```
Godina eq {closing year}
AND DelovodnaKnjigaKey eq {registry book key}
AND UsageStatus eq 'Pending'
```

If any `Pending` items remain, year closing must be blocked with a message listing the reserved numbers that must be resolved.

---

## AuditLog entries related to this list

| EventType | Severity | Trigger |
|---|---|---|
| `ReservationCreated` | Info | Admin creates a reservation |
| `ReservationCancelled` | Info | Admin manually cancels a `Pending` reservation |
| `ReservedNumberUsed` | Info | Step 11a: number marked `Used`; `UsedByItemId` written |
| `ReservedNumberDeletionFailed` | Warning | Step 11b: mark succeeded but delete failed; item left as cleanup |
| `ReservedNumberMarkFailed` | Critical | Step 11a: mark itself failed; `CounterSyncFailed` set |
| `NextNumberReserved` | Warning | Step 6 Branch A: sequential assignment blocked by reserved number |
| `ReservedAlreadyUsed` | Warning | Step 6 Branch B: reserved number already consumed in a previous run |

---

## Permissions

| Principal | Permission level |
|---|---|
| Service account (Power Automate) | Contribute (Create, Read, Update, Delete) |
| Admin / Consultant users | Read (for recovery panel display) |
| Registrar / all other users | No access |

End users must not be able to create, modify, or delete reservation items directly.

---

## Manual verification checklist (human developer)

Before wiring any flow to this list, confirm and return to Claude:

- [ ] List exists with the exact name chosen
- [ ] All columns created with correct types
- [ ] `RezervisaniBroj` has an index
- [ ] `DelovodnaKnjigaKey` has an index
- [ ] `Godina` has an index
- [ ] `UsageStatus` has exactly two choice values: `Pending`, `Used`
- [ ] `UsageStatus` default value = `Pending`
- [ ] `RezervisanoVreme` and `UsedAt` columns use **Date and Time** format with time zone **UTC**
- [ ] Service account connection reference has **Contribute** permission on this list
- [ ] End users do **not** have write permission on this list
- [ ] Internal column names confirmed

Return the following to Claude after verification:

```
List internal URL: 
RezervisaniBroj internal name: 
DelovodnaKnjigaKey internal name: 
Godina internal name: 
UsageStatus internal name: 
UsageStatus choice internal values (Pending / Used): 
UsedByItemId internal name: 
UsedAt internal name: 
RezervisaoEmail internal name: 
RezervisanoVreme internal name: 
Napomena internal name: 
```
