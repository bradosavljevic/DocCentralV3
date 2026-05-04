# Archive and year-closing rules: shared reference

## Stanje transition rules for archiving

| From | To | Flow | Allowed |
|---|---|---|---|
| Zavedeno | Arhivirano | CF_DocCentralV3_ArchiveDocument | YES |
| U odobravanju | Arhivirano | — | NO |
| Odobreno | Arhivirano | — | NO (requires client confirmation) |
| Odbijeno | Arhivirano | — | NO |
| Arhivirano | any | — | NO (terminal state) |

Direct archiving from `Zavedeno` is the only permitted path in v1.
All other transitions from non-Zavedeno states are rejected by ArchiveDocument with `INVALID_STATUS_FOR_ARCHIVE`.

---

## Archive fields on Svi predmeti (confirmed display names)

| Display name | Internal name | Type | Set by |
|---|---|---|---|
| Stanje | UNKNOWN | Choice or Text | ArchiveDocument |
| ArhivskiZnak | UNKNOWN | Text | ArchiveDocument |
| Arhivirano | UNKNOWN | Date | ArchiveDocument — display name confirmed |
| Arhivirao | UNKNOWN | Text (email) | ArchiveDocument — display name confirmed |

---

## Year-closing preconditions (all must pass — in order)

| # | Precondition | Checked against | Failure code |
|---|---|---|---|
| 1 | `yearToClose` equals App Config active year | App Config | YEAR_NOT_ACTIVE |
| 2 | Active year is not already closed | App Config closed flag | YEAR_ALREADY_CLOSED |
| 3 | No Svi predmeti documents for this year with Stanje ≠ Arhivirano | Svi predmeti (Top 1 filter) | DOCUMENTS_NOT_ALL_ARCHIVED |
| 4 | No items in RezervisaniBrojevi for this year | RezervisaniBrojevi (Top 1 filter) | RESERVED_NUMBERS_EXIST |

Checks 3 and 4 use `Top: 1` — existence check only, not full count retrieval.
This avoids loading large item sets into memory.

---

## Year-closing App Config operations

After all preconditions pass, two writes occur:

1. **PATCH existing registry book item** (If-Match with ETag)
   - Mark as closed
   - Record ClosedAt and ClosedBy

2. **CREATE new registry book item** for next year
   - Counter starts at 0 (or 1 — UNKNOWN, confirm from GenerateRegistryNumber flow)
   - IsClosed = false
   - Registry number format copied from closing year (UNKNOWN exact structure)

Whether a separate "active year pointer" item in App Config needs a third write depends on
the App Config schema — UNKNOWN. If the active year is determined by which registry book
is not closed, the create alone is sufficient.

---

## Irreversibility rule

Once a year is closed (closed flag set in App Config), it cannot be reopened.
This is enforced by:
- CloseRegistryYear Step 5 (YEAR_ALREADY_CLOSED pre-check)
- No unlock endpoint, no admin override, no hidden parameter

CreateDocument also enforces this by checking that the active year is open before allowing registration.

---

## Archive sign (ArhivskiZnak) validation

Valid archive signs are stored in App Config (UNKNOWN key / category).
ArchiveDocument reads them at the start of each run and validates per document.
The App Config read is done once per flow run — not once per document.

---

## ETag / If-Match usage

| Flow | Where used | Retry limit |
|---|---|---|
| ArchiveDocument | Per-document PATCH on Svi predmeti | 0 retries — 412 → re-read → idempotency check |
| CloseRegistryYear | PATCH on App Config registry book item | 0 retries — 412 → CONCURRENT_UPDATE |

ArchiveDocument 412 handling:
- Re-read item.
- If Stanje is already `Arhivirano` → treat as success (idempotent — same intent).
- Otherwise → add to `varFailures` (CONCURRENT_UPDATE), continue loop.

CloseRegistryYear 412 handling:
- Do not retry. A concurrent modification on the registry book config is unexpected.
- Return `CONCURRENT_UPDATE`. The Canvas App should prompt the user to refresh and recheck.

---

## Batch archiving limits

ArchiveDocument accepts an array of documents in one call.
Recommended maximum: 50 documents per call (UNKNOWN — confirm with client).
Each document is processed sequentially (Apply to each).
Failures are accumulated — other documents continue even if one fails.

---

## Audit log events

| Event | Flow | Severity | When |
|---|---|---|---|
| ArchiveDocument / Started | ArchiveDocument | Info | Before loop |
| ArchiveDocument / Success | ArchiveDocument | Info | Per document, on success |
| ArchiveDocument / Warning | ArchiveDocument | Warning | Per document, on validation failure |
| ArchiveDocument / Summary | ArchiveDocument | Info / Warning / Error | After loop |
| CloseRegistryYear / Started | CloseRegistryYear | Info | Before preconditions |
| CloseRegistryYear / Warning | CloseRegistryYear | Warning | Per failed precondition |
| CloseRegistryYear / Success | CloseRegistryYear | Info | After successful close |
| CloseRegistryYear / Error | CloseRegistryYear | Error | Catch scope |

---

## Open items (shared)

| Item | Status |
|---|---|
| Svi predmeti Stanje internal column name | UNKNOWN |
| Svi predmeti ArhivskiZnak internal column name | UNKNOWN |
| Svi predmeti Arhivirano internal column name | UNKNOWN — display name confirmed |
| Svi predmeti Arhivirao internal column name | UNKNOWN — display name confirmed |
| Svi predmeti year filter column (for CloseRegistryYear) | UNKNOWN |
| RezervisaniBrojevi year filter column | UNKNOWN |
| App Config key for archive signs list | UNKNOWN |
| App Config column for active year | UNKNOWN |
| App Config column for closed flag on registry book | UNKNOWN |
| App Config column for ClosedAt and ClosedBy | UNKNOWN |
| App Config counter initial value (0 or 1) | UNKNOWN — align with GenerateRegistryNumber |
| Whether active year pointer is separate App Config item | UNKNOWN |
| Whether registry number format copies or resets on new year | UNKNOWN |
| Maximum documents per ArchiveDocument call | UNKNOWN — suggest 50 |
| Whether Odobreno → Arhivirano will ever be required | TO BE CONFIRMED |
