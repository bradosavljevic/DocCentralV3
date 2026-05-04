# Test cases: CF_DocCentralV3_CloseRegistryYear

## Prerequisites

- App Config has a registry book entry for the active year (e.g. 2026) with IsClosed = false.
- All Svi predmeti items for 2026 are in Stanje = `Arhivirano` (for positive path).
- RezervisaniBrojevi list is empty for 2026 (for positive path).
- CF_DocCentralV3_LogEvent published.
- Admin access to reset App Config between tests.

---

## TC-001: All preconditions met — year closed successfully

**Setup:**
- App Config: active year = 2026, IsClosed = false.
- Svi predmeti: all documents for 2026 have Stanje = `Arhivirano`.
- RezervisaniBrojevi: no items for 2026.

**Input:**
```json
{ "yearToClose": 2026, "initiatorEmail": "admin@organizacija.rs", "correlationId": "tc-001" }
```

**Expected:**
- success = true. closedYear = 2026. newActiveYear = 2027.
- App Config: existing registry book item for 2026 — IsClosed = true, ClosedAt set, ClosedBy = `admin@organizacija.rs`.
- App Config: new registry book item created for 2027 — IsClosed = false, Counter = 0 (or 1), FormatString copied.
- AuditLog: Started + Success entries.

**Pass criteria:** Flow green. Both App Config writes confirmed. New year usable for document registration.

---

## TC-002: yearToClose does not match active year — YEAR_NOT_ACTIVE

**Setup:** App Config active year = 2026.

**Input:** `"yearToClose": 2025`

**Expected:**
- success = false. errorCode = `YEAR_NOT_ACTIVE`.
- failedPrecondition includes `Aktivna godina u sistemu je 2026`.
- App Config unchanged.
- AuditLog: Warning entry.

**Pass criteria:** Precondition 1 caught. No writes made.

---

## TC-003: Year already closed — YEAR_ALREADY_CLOSED

**Setup:** App Config registry book item for 2026 has IsClosed = true (manually set).

**Input:** `"yearToClose": 2026`

**Expected:**
- success = false. errorCode = `YEAR_ALREADY_CLOSED`.
- App Config unchanged.

**Pass criteria:** Precondition 2 caught. No PATCH attempted.

---

## TC-004: One document not archived — DOCUMENTS_NOT_ALL_ARCHIVED

**Setup:** Active year = 2026. One Svi predmeti item for 2026 has Stanje = `Zavedeno`.

**Input:** `"yearToClose": 2026`

**Expected:**
- success = false. errorCode = `DOCUMENTS_NOT_ALL_ARCHIVED`.
- failedPrecondition = `Postoje dokumenti koji nisu arhivirani.`
- App Config unchanged.

**Pass criteria:** Precondition 3 caught. Get items returns the one non-archived item (Top 1 sufficient).

---

## TC-005: Reserved numbers exist — RESERVED_NUMBERS_EXIST

**Setup:** Active year = 2026. All documents archived. RezervisaniBrojevi has 1 item for 2026.

**Input:** `"yearToClose": 2026`

**Expected:**
- success = false. errorCode = `RESERVED_NUMBERS_EXIST`.
- failedPrecondition = `Postoje rezervisani brojevi za ovu godinu.`

**Pass criteria:** Precondition 4 caught after preconditions 1–3 pass.

---

## TC-006: Both documents and reserved numbers exist — documents caught first

**Setup:** Both TC-004 and TC-005 conditions present simultaneously.

**Expected:**
- errorCode = `DOCUMENTS_NOT_ALL_ARCHIVED` (precondition 3 checked before precondition 4).

**Pass criteria:** Correct precondition ordering. First failure returned.

---

## TC-007: 412 conflict on PATCH — CONCURRENT_UPDATE

**Method:**
1. Read App Config registry book ETag.
2. Manually update any field to force ETag change.
3. Call CloseRegistryYear — flow reads old ETag, PATCH fails with 412.

**Expected:**
- success = false. errorCode = `CONCURRENT_UPDATE`.
- AuditLog: Error entry.
- App Config: year NOT closed (PATCH failed). No new registry book created.

**Pass criteria:** 412 handled. CONCURRENT_UPDATE returned. No partial state.

---

## TC-008: Two admins submit simultaneously — one wins, one gets CONCURRENT_UPDATE or YEAR_ALREADY_CLOSED

**Method:** Run TC-001 twice simultaneously (parallel browser tabs).

**Expected:**
- First call: success = true (wins the race, PATCH succeeds).
- Second call:
  - If second PATCH reaches SharePoint before first completes: 412 → CONCURRENT_UPDATE.
  - If second PATCH reaches after first completes and re-reads: YEAR_ALREADY_CLOSED (IsClosed = true).

**Pass criteria:** No double-close. At most one new registry book created for 2027.

---

## TC-009: Precondition order — YEAR_NOT_ACTIVE checked before reading Svi predmeti

**Setup:** yearToClose = 2025 (wrong year). Svi predmeti has non-archived documents for 2025.

**Expected:**
- errorCode = `YEAR_NOT_ACTIVE` — returned without querying Svi predmeti.

**Pass criteria:** App Config read is the only SharePoint call made.

---

## TC-010: New registry book fields verified

**After TC-001:** Inspect the newly created App Config item for 2027.

**Expected:**
- FormatString = same value as 2026 registry book.
- Counter = 0 (or configured start value).
- IsClosed = false.
- ActiveYear = 2027.

**Pass criteria:** All fields on new registry book correct.

---

## TC-011: GenerateRegistryNumber works after year close

**After TC-001:** Attempt to register a new document.

**Expected:**
- CF_DocCentralV3_GenerateRegistryNumber reads the 2027 App Config item.
- New registry number format: UNKNOWN (should use 2027 format).
- Document created successfully.

**Pass criteria:** New year is functional for document registration without manual App Config adjustment.

---

## TC-012: correlationId auto-generated when empty

**Input:** `"correlationId": ""`

**Expected:** response correlationId is a valid GUID. All AuditLog entries use the same GUID.

---

## Post-test cleanup

1. If TC-001 ran: reset App Config — delete 2027 registry book item; reopen 2026 book (set IsClosed = false, clear ClosedAt/ClosedBy). **Do this in SharePoint directly — no flow can reopen a closed year.**
2. Delete AuditLog entries created during testing.
3. Confirm flow is inside DocCentralV3 solution.

**Important:** After TC-001, CreateDocument will attempt to use the 2027 registry book. Reset App Config promptly if running further tests in 2026.
