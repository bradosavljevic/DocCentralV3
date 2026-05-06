# SharePoint Guide: NumberLocks / ZakljucavanjeBrojeva

Last updated: 2026-05-06

---

## Purpose

`NumberLocks` is the concurrency control mechanism for registry number (`DelovodniBroj`) assignment. It prevents two parallel flow runs from reading the same "next number" simultaneously, which would produce a duplicate or skipped number.

SharePoint does not provide true database-level locking. This list implements a **create-then-validate** pattern: the flow attempts to create a lock item with a unique `LockKey`. Because `LockKey` has a unique column constraint enforced at the SharePoint list level, only one flow run can succeed in creating that item. All other concurrent runs receive a SharePoint error on the create action, which the flow treats as "lock already held — fail fast." This is the only safe concurrency pattern available in Power Automate + SharePoint without a dedicated database.

---

## List name

Use exactly one of the following names. Confirm which name is used and return it to Claude before flow guides reference the list.

- `NumberLocks` (English, recommended)
- `ZakljucavanjeBrojeva` (Serbian alternative)

---

## Required columns

| Display name | Suggested internal name | Type | Required | Notes |
|---|---|---|---|---|
| Title | Title | Single line text | Yes | Set equal to LockKey value for readability |
| LockKey | LockKey | Single line text | Yes | **Unique + Indexed** — see constraints section |
| LockedBy | LockedBy | Single line text | No | Email of the service account / flow run identifier |
| LockedAt | LockedAt | Date and Time | Yes | UTC timestamp of lock creation |
| ExpiresAt | ExpiresAt | Date and Time | Yes | UTC timestamp of lock expiry (LockedAt + 5 minutes) |
| FlowRunId | FlowRunId | Single line text | No | Power Automate workflow run ID for correlation |
| Status | Status | Choice | Yes | See status values below |
| RelatedItemId | RelatedItemId | Number | No | SharePoint ID of the `Svi predmeti` item being created |
| RelatedDocumentId | RelatedDocumentId | Number | No | SharePoint ID of the `Dokumenta` item being created |
| ErrorCode | ErrorCode | Single line text | No | Machine-readable error code written on failure |
| ErrorDetail | ErrorDetail | Multiple lines of text | No | Technical error detail for AuditLog/recovery |

---

## Status choice values

| Value | Meaning |
|---|---|
| `Active` | Lock is currently held; flow is in progress |
| `Released` | Flow completed successfully; lock released |
| `Expired` | Lock passed its ExpiresAt timestamp without being released or failed |
| `Failed` | Flow failed; lock left in failed state; recovery required |

---

## Required constraints

### LockKey — Unique column constraint

1. In SharePoint list settings → Column: `LockKey`
2. Enable **Enforce unique values: Yes**
3. This constraint is the foundation of the entire locking mechanism. Without it the pattern is unsafe.

### LockKey — Index

1. In SharePoint list settings → Indexed columns
2. Add index on `LockKey`
3. This is required for OData filter performance in recovery queries

### Recommended additional indexes

| Column | Reason |
|---|---|
| Status | Recovery queries filter by `Status eq 'Active'` or `Status eq 'Failed'` |
| ExpiresAt | Expiry cleanup queries filter by `ExpiresAt lt [now]` |

---

## LockKey format

The `LockKey` value must be unique per registry book and year. Recommended format:

```
{RegistryBookKey}_{Year}
```

Examples:
- `DELOVODNA_2026`
- `BOOK01_2026`

This ensures that different registry books do not block each other while still preventing parallel assignment within the same book/year combination.

---

## Lock duration

- Lock duration: **5 minutes**
- `ExpiresAt = LockedAt + 5 minutes` (calculated in UTC by the flow)
- An expired lock (`Status = Active` AND `ExpiresAt < utcNow()`) does **not** automatically allow a new flow run to proceed
- Expired lock requires recovery validation before any new number can be assigned
- Only admin/consultant or the recovery flow may mark an expired lock as `Expired` or `Released` after confirming the transaction state

---

## Lock lifecycle

```
[Flow run starts]
        │
        ▼
 Create NumberLocks item
 with Status = Active
        │
   ┌────┴────────────────────┐
   │ Success                 │ Conflict (duplicate LockKey)
   ▼                         ▼
Lock acquired            Lock already held →
Continue flow            Fail fast, return
                         "system busy" error
        │
   ┌────┴──────────────────────────────┐
   │ Flow completes successfully       │ Flow fails
   ▼                                   ▼
Update Status = Released          Update Status = Failed
Delete lock item (optional         Write ErrorCode + ErrorDetail
cleanup) or leave Released         Do NOT delete item
                                   Trigger AuditLog Critical entry
                                   Return FailedNeedsRecovery
```

---

## Retention

- `Released` lock items may be deleted after 24 hours (scheduled cleanup)
- `Failed` lock items must be retained until recovery is confirmed and the item is manually or programmatically resolved
- `Expired` lock items must be retained until recovery validates the transaction state

---

## Manual verification checklist (human developer)

Before wiring any flow to this list, confirm and return to Claude:

- [ ] List exists with the exact name you chose (`NumberLocks` or `ZakljucavanjeBrojeva`)
- [ ] All columns created with correct types
- [ ] `LockKey` has **Enforce unique values = Yes**
- [ ] `LockKey` has an index
- [ ] `Status` has all four choice values: `Active`, `Released`, `Expired`, `Failed`
- [ ] `LockedAt` and `ExpiresAt` columns use **Date and Time** format (not Date only), with time zone **UTC**
- [ ] Service account connection reference has **Contribute** permission on this list
- [ ] End users do **not** have any write permission on this list
- [ ] Internal column names confirmed (return the actual internal names, not display names)

Return the following to Claude after verification:

```
List internal URL: 
LockKey internal name: 
Status internal name: 
Status choice internal values: 
LockedAt internal name: 
ExpiresAt internal name: 
FlowRunId internal name: 
RelatedItemId internal name: 
RelatedDocumentId internal name: 
ErrorCode internal name: 
ErrorDetail internal name: 
```
