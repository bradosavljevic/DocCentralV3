# Test cases: CF_DocCentralV3_UseReservedNumber

## Prerequisites before testing

- RezervisaniBrojevi list has at least one test item with a known delovodniBroj and year.
- Svi predmeti list exists (may be empty for most tests).
- App Config has an active year entry.
- CF_DocCentralV3_LogEvent is built and working.
- All UNKNOWN column names in `reserved-number-field-mapping.md` have been resolved.

## How to test

Create a temporary instant cloud flow with a "Run a Child Flow" action targeting
CF_DocCentralV3_UseReservedNumber. Provide inputs manually.

After each test, verify:
1. Flow run history for CF_DocCentralV3_UseReservedNumber.
2. AuditLog list — check event entries.
3. RezervisaniBrojevi list — confirm the item was NOT deleted by this flow (deletion belongs to the parent).

---

## TC-001: Happy path — valid reserved number, current year

**Purpose:** Verify that a valid reserved number passes all checks and returns success.

**Setup:** Ensure a RezervisaniBrojevi item exists with:
- delovodniBroj = `15/2026`
- year = `2026`
- Item ID = noted as `<testItemId>`

Active year in App Config = `2026`.
No item in Svi predmeti with DelovodniBroj = `15/2026`.

**Input:**
```json
{
  "reservedNumberId": <testItemId>,
  "requestedYear": 2026,
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-001"
}
```

**Expected response:**
```json
{
  "success": true,
  "delovodniBroj": "15/2026",
  "reservedNumberId": <testItemId>,
  "filingDateSuggestion": "<value or empty>",
  "correlationId": "test-correlation-001",
  "message": "",
  "errorCode": ""
}
```

**AuditLog entries expected:**
- 1 × UseReservedNumber / Info / Started
- 1 × UseReservedNumber / Info / Success

**RezervisaniBrojevi after test:** Item still exists (NOT deleted — deletion is the parent's job).

**Pass criteria:** Flow green. Correct delovodniBroj returned. Item not deleted.

---

## TC-002: Reserved number ID does not exist

**Purpose:** Verify RESERVED_NUMBER_NOT_FOUND when the item is missing.

**Input:**
```json
{
  "reservedNumberId": 99999,
  "requestedYear": 2026,
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-002"
}
```

Item ID 99999 must not exist in RezervisaniBrojevi.

**Expected response:**
```json
{
  "success": false,
  "delovodniBroj": "",
  "reservedNumberId": 0,
  "filingDateSuggestion": "",
  "errorCode": "RESERVED_NUMBER_NOT_FOUND",
  "correlationId": "test-correlation-002"
}
```

**AuditLog:** 1 × UseReservedNumber / Error / Failed / RESERVED_NUMBER_NOT_FOUND.

**Pass criteria:** Flow returns structured failure. No unhandled exception.

---

## TC-003: Year mismatch — reserved number is from a different year

**Purpose:** Verify RESERVED_NUMBER_YEAR_MISMATCH when the reserved number's year does not match requestedYear.

**Setup:** RezervisaniBrojevi item with year = `2025`, ID = `<testItemId2025>`.

**Input:**
```json
{
  "reservedNumberId": <testItemId2025>,
  "requestedYear": 2026,
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-003"
}
```

**Expected:**
```json
{
  "success": false,
  "errorCode": "RESERVED_NUMBER_YEAR_MISMATCH",
  "correlationId": "test-correlation-003"
}
```

**AuditLog:** 1 × UseReservedNumber / Warning / Failed / RESERVED_NUMBER_YEAR_MISMATCH.

**Pass criteria:** Correct error code returned. RezervisaniBrojevi item not deleted.

---

## TC-004: Requested year is not the active year in App Config

**Purpose:** Verify YEAR_NOT_ACTIVE when requestedYear does not match App Config active year.

**Setup:**
- Active year in App Config = `2026`.
- RezervisaniBrojevi item exists with year = `2027`, ID = `<testItemId2027>`.

**Input:**
```json
{
  "reservedNumberId": <testItemId2027>,
  "requestedYear": 2027,
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-004"
}
```

Item year (2027) matches requestedYear (2027) — passes the first year check.
But requestedYear (2027) does not match App Config active year (2026) — fails second check.

**Expected:**
```json
{
  "success": false,
  "errorCode": "YEAR_NOT_ACTIVE",
  "correlationId": "test-correlation-004"
}
```

**AuditLog:** 1 × UseReservedNumber / Warning / Failed / YEAR_NOT_ACTIVE.

**Pass criteria:** YEAR_NOT_ACTIVE returned, not RESERVED_NUMBER_YEAR_MISMATCH. Both validations are distinct.

---

## TC-005: Reserved number already used — document exists in Svi predmeti

**Purpose:** Verify RESERVED_NUMBER_ALREADY_USED and Critical log when a document already carries the same delovodniBroj.

**Setup:**
- RezervisaniBrojevi item with delovodniBroj = `20/2026`, year = `2026`, ID = `<testItemId>`.
- Svi predmeti item already exists with DelovodniBroj = `20/2026` (manually created for test).

**Input:**
```json
{
  "reservedNumberId": <testItemId>,
  "requestedYear": 2026,
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-005"
}
```

**Expected:**
```json
{
  "success": false,
  "errorCode": "RESERVED_NUMBER_ALREADY_USED",
  "correlationId": "test-correlation-005"
}
```

**AuditLog:** 1 × UseReservedNumber / Critical / Failed / RESERVED_NUMBER_ALREADY_USED.

**Post-test cleanup:** Delete the manually created Svi predmeti item.

**Pass criteria:** Critical severity entry in AuditLog. Flow returns structured failure. No exception.

---

## TC-006: Empty correlationId — auto-generation

**Purpose:** Verify that an auto-generated correlationId GUID is returned and used in AuditLog entries.

**Input:** Same as TC-001 but with `"correlationId": ""`.

**Expected:** Response correlationId is a non-empty GUID (36 chars, `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` format).
Both AuditLog entries (Started and Success) share the same auto-generated correlationId.

**Pass criteria:** Valid GUID returned. AuditLog entries linked by that GUID.

---

## TC-007: Verify item is NOT deleted after this flow

**Purpose:** Confirm that CF_DocCentralV3_UseReservedNumber never deletes the RezervisaniBrojevi item.

**Method:** Run TC-001. After the flow returns success, open RezervisaniBrojevi in SharePoint.
Confirm the item is still present.

**Pass criteria:** Item exists in RezervisaniBrojevi after a successful flow run.

---

## TC-008: Double submission simulation

**Purpose:** Verify that submitting the same reservedNumberId twice in quick succession
does not produce two successful responses.

**Method:**
1. Note a valid reservedNumberId.
2. Run TC-001.
3. Before the parent flow would normally delete the item, run TC-001 again with the same reservedNumberId.

Since the item is not deleted (this flow does not delete), both calls will succeed
(the item still exists and no Svi predmeti document exists for it).
This is expected — the double-submission guard is at the Svi predmeti duplicate check level,
which only activates once the parent flow creates the item.

The real guard against double-use is that CF_DocCentralV3_CreateDocument deletes the item
after the first successful use, causing the second attempt's TC-002 scenario.

**Pass criteria:** Both calls return success (expected — deletion has not happened yet).
Document this result as confirmation that deletion in the parent flow is the true guard.

---

## TC-009: Large delovodniBroj string

**Purpose:** Verify that an unusually long delovodniBroj value is handled without truncation.

**Setup:** Create a RezervisaniBrojevi item with a 100-character delovodniBroj value.

**Expected:** Flow succeeds. Full delovodniBroj returned in response.

**Pass criteria:** Response delovodniBroj matches the full 100-character value.

---

## Post-test cleanup

1. Delete any Svi predmeti items created manually for TC-005.
2. Leave RezervisaniBrojevi test items in place for re-use in integration testing with CF_DocCentralV3_CreateDocument.
3. Restore App Config active year if changed during testing.
4. Confirm the flow is inside the DocCentralV3 solution.
