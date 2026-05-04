# Test cases: CF_DocCentralV3_ArchiveDocument

## Prerequisites

- At least one Svi predmeti item with Stanje = `Zavedeno`.
- App Config populated with at least one valid archive sign (UNKNOWN key).
- CF_DocCentralV3_LogEvent published.
- Two test users available (for concurrent update test).

---

## TC-001: Single document, valid state and valid sign — full success

**Setup:** Item with Stanje = `Zavedeno`.

**Input:**
```json
{
  "documentsJson": "[{\"documentItemId\":<id>,\"delovodniBroj\":\"5/2026\",\"arhivskiZnak\":\"1.1\"}]",
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "tc-001"
}
```

**Expected:**
- success = true. archivedCount = 1. failedCount = 0. failures = `[]`.
- Svi predmeti: Stanje = `Arhivirano`. ArhivskiZnak = `1.1`. Arhivirano = today's date. Arhivirao = `test.user@organizacija.rs`.
- AuditLog: Started + per-document Success + overall Success.

**Pass criteria:** Flow green. All four fields updated. Audit log correct.

---

## TC-002: Multiple documents, all valid — all succeed

**Setup:** 3 items, all Stanje = `Zavedeno`.

**Input:** documentsJson with 3 entries, each with a valid archive sign.

**Expected:**
- success = true. archivedCount = 3. failedCount = 0.
- All 3 items updated in Svi predmeti.

**Pass criteria:** All 3 archived. Response correct.

---

## TC-003: One document has invalid Stanje — partial failure

**Setup:** 2 items — item A: Stanje = `Zavedeno`, item B: Stanje = `U odobravanju`.

**Input:** documentsJson with both items.

**Expected:**
- success = false. archivedCount = 1. failedCount = 1.
- failures contains 1 entry: item B, errorCode = `INVALID_STATUS_FOR_ARCHIVE`, message includes `U odobravanju`.
- Item A archived correctly. Item B unchanged.

**Pass criteria:** Partial result. Correct failure entry. Item B untouched.

---

## TC-004: Invalid archive sign — per-document failure

**Setup:** Item with Stanje = `Zavedeno`.

**Input:** `"arhivskiZnak": "INVALID_CODE"` (not in App Config).

**Expected:**
- success = false. failedCount = 1.
- failures[0].errorCode = `INVALID_ARCHIVE_SIGN`.
- Item unchanged in Svi predmeti (no PATCH attempted).
- Validation happens before Get item.

**Pass criteria:** INVALID_ARCHIVE_SIGN returned. No PATCH called for this document.

---

## TC-005: Document not found — DOC_NOT_FOUND

**Input:** `"documentItemId": 99999` (non-existent item).

**Expected:**
- success = false. failedCount = 1.
- failures[0].errorCode = `DOC_NOT_FOUND`.

**Pass criteria:** DOC_NOT_FOUND returned. No exception in flow.

---

## TC-006: Empty documents array — NO_DOCUMENTS_PROVIDED

**Input:** `"documentsJson": "[]"`

**Expected:**
- Immediate failure before Log Started and before loop.
- Response: success = false, errorCode = `NO_DOCUMENTS_PROVIDED`.
- AuditLog: Warning entry for NO_DOCUMENTS_PROVIDED.

**Pass criteria:** Flow returns immediately. Loop not entered.

---

## TC-007: 412 conflict — document already Arhivirano (idempotent)

**Method:**
1. Set item Stanje = `Zavedeno`.
2. While flow is processing (simulate by running two calls simultaneously), manually set item to `Arhivirano` before PATCH.
3. Easier simulation: call ArchiveDocument on an item that is already `Arhivirano`.

**Expected:**
- 412 returned by PATCH.
- Re-read detects Stanje = `Arhivirano`.
- Flow treats as success — archivedCount incremented.
- success = true (if only document in batch).

**Pass criteria:** Idempotent — already-archived document counts as success. No failure entry.

---

## TC-008: 412 conflict — document modified to different state

**Method:** Simulate 412 where re-read shows Stanje ≠ `Arhivirano` (e.g., somehow changed to `Odobreno`).
(Artificially construct by: (1) reading ETag, (2) manually changing item, (3) calling flow which will then 412.)

**Expected:**
- CONCURRENT_UPDATE in failures array.
- failedCount = 1.

**Pass criteria:** CONCURRENT_UPDATE returned. No retry attempted.

---

## TC-009: Mixed batch — some valid, some invalid sign, some not found

**Setup:** 3 items:
- A: valid, valid sign.
- B: valid, invalid sign.
- C: non-existent ID.

**Expected:**
- success = false. archivedCount = 1. failedCount = 2.
- failures[0]: B, INVALID_ARCHIVE_SIGN.
- failures[1]: C, DOC_NOT_FOUND.
- Item A archived.

**Pass criteria:** All 3 processed independently. Correct per-document results.

---

## TC-010: Arhivirao field written correctly

**Input:** `"initiatorEmail": "specific.user@organizacija.rs"`

**Expected:**
- Archived item: Arhivirao = `specific.user@organizacija.rs`.
- Arhivirano date is today (UTC).

**Pass criteria:** Both display-name-confirmed fields set exactly as expected.

---

## TC-011: AuditLog severity — partial failure

**Setup:** 2 items, 1 valid, 1 invalid sign.

**Expected:**
- Final overall log entry has severity = `Warning`.
- Message: `Arhiviranje završeno. Uspešno: 1, Neuspešno: 1.`

**Pass criteria:** Warning severity on partial failure. Message counts correct.

---

## TC-012: AuditLog severity — full failure

**Setup:** 2 items, both invalid sign.

**Expected:**
- Final overall log entry has severity = `Error`.
- archivedCount = 0.

**Pass criteria:** Error severity when nothing succeeds.

---

## TC-013: correlationId auto-generated when not provided

**Input:** `"correlationId": ""`

**Expected:**
- Response correlationId is a valid GUID (auto-generated).
- AuditLog entries all reference the same auto-generated GUID.

**Pass criteria:** Consistent correlationId across all log entries.

---

## TC-014: App Config archive signs not configured (empty list)

**Setup:** Temporarily remove or filter out all archive signs from App Config.

**Expected:**
- Log Warning about unconfigured signs.
- All documents fail with INVALID_ARCHIVE_SIGN (no valid signs to match against).
- success = false.

**Pass criteria:** Flow does not crash. Warning logged. Failure entries correct.

---

## Post-test cleanup

1. Reset test Svi predmeti items: set Stanje back to `Zavedeno`, clear Arhivirano/Arhivirao/ArhivskiZnak.
2. Delete test AuditLog entries.
3. Confirm flow is inside DocCentralV3 solution.
