# Test cases: CF_DocCentralV3_GenerateRegistryNumber

## Prerequisites before testing

- App Config list has exactly one active registry book row with valid counter, active year, and format string.
- AuditLog list exists.
- Svi predmeti list exists (may be empty for most tests).
- CF_DocCentralV3_LogEvent is built and working.
- All App Config UNKNOWN keys have been resolved and the flow has been updated.

## How to test

Create a temporary instant cloud flow with a "Run a Child Flow" action targeting
CF_DocCentralV3_GenerateRegistryNumber. Provide test inputs manually.

After each test, verify:
1. Flow run history for CF_DocCentralV3_GenerateRegistryNumber.
2. AuditLog list — check new entries.
3. App Config list — check that the counter was (or was not) incremented.
4. Svi predmeti list — check for duplicate entries if relevant.

---

## TC-001: Happy path — correlationId provided

**Purpose:** Verify that a valid registry number is generated and the counter incremented.

**Input:**
```json
{
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-001"
}
```

**Expected response:**
```json
{
  "success": true,
  "delovodniBroj": "<counter+1>/<activeYear>",
  "counterValue": <currentCounter + 1>,
  "activeYear": <activeYear from App Config>,
  "correlationId": "test-correlation-001",
  "message": "",
  "errorCode": ""
}
```

**App Config state after test:**
- Counter column = original counter + 1.

**AuditLog entries expected:**
- 1 × GenerateRegistryNumber / Info / Started
- 1 × GenerateRegistryNumber / Info / Success

**Pass criteria:** Flow run green. Counter incremented by exactly 1. delovodniBroj contains the new counter value and active year.

---

## TC-002: Empty correlationId — auto-generation

**Purpose:** Verify the flow generates a GUID when correlationId is empty.

**Input:**
```json
{
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": ""
}
```

**Expected:** Response `correlationId` is a non-empty GUID (36 chars).
Both Started and Success AuditLog entries share the same auto-generated correlationId.

**Pass criteria:** correlationId in response is a valid GUID. Counter incremented.

---

## TC-003: Sequential calls — counter increments correctly

**Purpose:** Verify that 3 sequential calls each produce a unique, incrementing delovodniBroj.

**Method:** Run TC-001 three times in sequence. Record delovodniBroj for each call.

**Expected:** Three distinct delovodniBroj values with consecutive counter values.
Example: 42/2026, 43/2026, 44/2026.

**Pass criteria:** No duplicate numbers. Counter incremented 3 times in total. 3 AuditLog Success entries.

---

## TC-004: Concurrent calls — no duplicates

**Purpose:** Verify that simultaneous calls do not produce duplicate registry numbers.

**Method:**
1. Note current counter value (e.g. 50).
2. Trigger 5 simultaneous calls using 5 parallel branches in a test flow.
3. Collect the 5 delovodniBroj values returned.

**Expected:**
- 5 unique delovodniBroj values (e.g. 51/2026 through 55/2026).
- App Config counter = original + 5.
- AuditLog contains Retried entries (Warning severity) for any flows that experienced ETag conflicts.

**Pass criteria:** All 5 values distinct. Counter incremented exactly 5 times total. No REGISTRY_NUMBER_GENERATION_FAILED errors.

---

## TC-005: Max retries exhausted (simulated)

**Purpose:** Verify the flow returns a structured failure when the retry limit is exceeded.

**Simulation approach:**
- Set the max retries App Config value to `1`.
- Trigger 3 simultaneous calls.
- At least one or two will exhaust retries.

**Expected for failing calls:**
```json
{
  "success": false,
  "delovodniBroj": "",
  "counterValue": 0,
  "activeYear": 0,
  "message": "Max retries exceeded — ETag conflict not resolved.",
  "errorCode": "REGISTRY_NUMBER_GENERATION_FAILED"
}
```

**AuditLog:** Retried (Warning) entry, then Failed (Error) entry for each exhausted flow.

**Restore:** Reset max retries in App Config to 5 after test.

**Pass criteria:** Failure response is structured (not an unhandled exception). At least one call succeeds.

---

## TC-006: App Config registry book row missing

**Purpose:** Verify REGISTRY_BOOK_NOT_FOUND error when App Config has no active registry row.

**Simulation approach:**
- Temporarily rename or delete the App Config filter key value for the active registry book.
  (Easiest: change the Key column value so the filter does not match.)

**Input:** Any valid input.

**Expected:**
```json
{
  "success": false,
  "errorCode": "REGISTRY_BOOK_NOT_FOUND",
  "message": "Registry book not found in App Config."
}
```

**AuditLog:** 1 × PowerAutomateError / Error / Failed with errorCode REGISTRY_BOOK_NOT_FOUND.

**Restore:** Revert the App Config key value after test.

**Pass criteria:** Flow returns structured failure. Counter not modified. No unhandled exception.

---

## TC-007: Duplicate safety check triggered

**Purpose:** Verify the REGISTRY_NUMBER_DUPLICATE guard works.

**Setup:**
1. Note the next counter value that the flow would produce (e.g. counter = 60, so next would be 61/2026).
2. Manually create an item in Svi predmeti with DelovodniBroj = `61/2026`.
3. Run the flow normally.

**Expected:**
- PATCH succeeds (counter updated to 61).
- Safety check finds the existing Svi predmeti item.
- Flow returns:
  ```json
  {
    "success": false,
    "errorCode": "REGISTRY_NUMBER_DUPLICATE",
    "message": "Duplicate registry number detected."
  }
  ```
- AuditLog: 1 × SystemError / Critical entry with REGISTRY_NUMBER_DUPLICATE errorCode.

**Post-test cleanup:**
- Delete the manually created Svi predmeti item.
- Note that App Config counter is now at 61 — the counter was consumed even though the flow failed. This is by design. The next call will use 62.

**Pass criteria:** Structured failure response. Critical log entry exists. Counter was incremented (accepted as "used").

---

## TC-008: Large initiatorEmail value

**Purpose:** Verify the flow handles long email addresses without truncation errors.

**Input:**
```json
{
  "initiatorEmail": "very.long.user.name.that.approaches.limit@very.long.domain.name.organizacija.rs",
  "correlationId": "test-correlation-008"
}
```

**Expected:** Flow succeeds. AuditLog entry stores the full email.

**Pass criteria:** Flow green. AuditLog UserEmail column contains full value.

---

## TC-009: Verify AuditLog entries for retry scenario

**Purpose:** Verify that all expected AuditLog entries are written for a retried call.

**Method:** Use TC-005 setup (low max retries, concurrent calls). Pick one call that experienced at least one 412 before succeeding.

**Expected AuditLog sequence for that flow run (same correlationId):**
1. GenerateRegistryNumber / Info / Started
2. GenerateRegistryNumber / Warning / Retried (one or more)
3. GenerateRegistryNumber / Info / Success

**Pass criteria:** All three entry types present. Started and Retried entries precede Success. All share the same correlationId.

---

## TC-010: Year rollover (manual verification)

**Purpose:** Verify the format string produces the correct delovodniBroj after the active year is updated in App Config.

**Setup:** Temporarily change the active year in App Config to `2099`.

**Expected:** delovodniBroj contains `2099` as the year component.

**Restore:** Revert active year after test.

**Pass criteria:** Format string correctly picks up the updated year value from App Config.

---

## Post-test cleanup

1. Reset App Config counter, max retries, and active year to production values if changed.
2. Restore App Config filter key value if changed.
3. Delete any manually created Svi predmeti test items.
4. Optionally filter or delete test AuditLog entries.
5. Confirm the flow is inside the DocCentralV3 solution.
