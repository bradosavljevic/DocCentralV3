# SharePoint Indexes and Unique Constraints

Last updated: 2026-05-06

---

## Required — Unique constraints (Enforce unique values = Yes)

| List / Library | Column | Rationale |
|---|---|---|
| `Svi predmeti` | `DelovodniBrojText` | Composite-encoded registry number; must be globally unique across all items |
| `NumberLocks` | `LockKey` | Atomic lock creation depends on this constraint — without it the create-then-validate pattern is unsafe |

### Why DelovodniBrojNumber is NOT unique

`DelovodniBrojNumber` (integer) repeats across years (number 1 is valid in 2025 and again in 2026) and across registry books. Setting a unique constraint on this column would reject valid records. Uniqueness is enforced through `DelovodniBrojText`, which encodes `{RegistryBookKey}-{Number}/{Year}` and is globally unique.

### Why Rezervisani brojevi.RezervisaniBroj is NOT unique

The same integer can be reserved in different years or for different registry books. Uniqueness of a reservation is logical: `(RezervisaniBroj, DelovodnaKnjigaKey, Godina)` must not duplicate. This is enforced by the admin creation flow (query before create), not by a SharePoint column constraint.

---

## Required — Indexed columns

### Svi predmeti

| Column | Reason |
|---|---|
| `DelovodniBrojText` | Duplicate detection query in step 5 |
| `DelovodniBrojNumber` | Sequence queries in step 4: `max(DelovodniBrojNumber)` grouped by book+year |
| `DelovodniBrojGodina` | Year-scoped queries |
| `DelovodnaKnjigaKey` | Registry book scoped queries |
| `TechnicalStatus` | FailedNeedsRecovery block check (step 2); recovery panel queries |

### Dokumenta

| Column | Reason |
|---|---|
| `DelovodniBrojText` | Metadata sync query by formatted number |
| `DelovodniBrojNumber` | Metadata sync and reporting queries |
| `DelovodnaKnjigaKey` | Registry book scoped queries |
| `DelovodniBrojGodina` | Year-scoped queries |
| `Attachment` | Separates main documents from attachments in view queries |
| `TechnicalStatus` | Sync validation queries |

### Rezervisani brojevi

| Column | Reason |
|---|---|
| `RezervisaniBroj` | Step 6 filter: `RezervisaniBroj eq {number}` |
| `DelovodnaKnjigaKey` | Step 6 filter: scoped to registry book |
| `Godina` | Step 6 filter: scoped to year; year-closing check |
| `UsageStatus` | Year-closing check: filter `UsageStatus eq 'Pending'`; cleanup queries |

### NumberLocks

| Column | Reason |
|---|---|
| `LockKey` | Step 1 unique create; step 1 failure branch query |
| `Status` | Recovery queries: `Status eq 'Active'`; cleanup queries |
| `ExpiresAt` | Expiry detection: `ExpiresAt lt utcNow()` |

### AuditLog

| Column | Reason |
|---|---|
| `Created` | Date-range queries for log review and cleanup |
| `EventType` | Filter by event type in admin log viewer |
| `RelatedItemId` | Correlate log entries with a specific `Svi predmeti` item |

---

## Recommended — Additional indexes (lower priority)

### Svi predmeti

| Column | Reason |
|---|---|
| `Stanje` | View filters by business status |
| `DatumZavodjenja` | Date-range views |
| `TipDokumenta` | Document type views and process resolution |
| `OrganizacionaJedinica` | Org-unit scoped views |

### Partneri

| Column | Reason |
|---|---|
| `PIB` | Partner lookup by tax ID |

---

## Summary table — all constraints

| List / Library | Column | Index | Unique |
|---|---|---|---|
| `Svi predmeti` | `DelovodniBrojText` | Yes | **Yes** |
| `Svi predmeti` | `DelovodniBrojNumber` | Yes | No |
| `Svi predmeti` | `DelovodniBrojGodina` | Yes | No |
| `Svi predmeti` | `DelovodnaKnjigaKey` | Yes | No |
| `Svi predmeti` | `TechnicalStatus` | Yes | No |
| `Dokumenta` | `DelovodniBrojText` | Yes | No |
| `Dokumenta` | `DelovodniBrojNumber` | Yes | No |
| `Dokumenta` | `DelovodnaKnjigaKey` | Yes | No |
| `Dokumenta` | `DelovodniBrojGodina` | Yes | No |
| `Dokumenta` | `Attachment` | Yes | No |
| `Dokumenta` | `TechnicalStatus` | Yes | No |
| `Rezervisani brojevi` | `RezervisaniBroj` | Yes | No |
| `Rezervisani brojevi` | `DelovodnaKnjigaKey` | Yes | No |
| `Rezervisani brojevi` | `Godina` | Yes | No |
| `Rezervisani brojevi` | `UsageStatus` | Yes | No |
| `NumberLocks` | `LockKey` | Yes | **Yes** |
| `NumberLocks` | `Status` | Yes | No |
| `NumberLocks` | `ExpiresAt` | Yes | No |
| `AuditLog` | `Created` | Yes | No |
