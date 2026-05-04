# Test cases: CF_DocCentralV3_CreateDocument

## Prerequisites before testing

- All 4 foundation child flows built and published: LogEvent, GenerateRegistryNumber, UseReservedNumber, AssignPermissions.
- Svi predmeti list exists with all confirmed column names.
- Dokumenti library exists with all confirmed column names.
- RezervisaniBrojevi has at least one test item.
- App Config has: active year entry, at least one document type entry, default org unit group.
- All UNKNOWN column names in `metadata-field-mapping.md` resolved.
- A small test PDF file Base64-encoded and available for input.
- CF_DocCentralV3_LogEvent working — verify AuditLog entries after each test.

## How to test

Create a temporary instant cloud flow with "Run a Child Flow" targeting CF_DocCentralV3_CreateDocument.
Provide inputs manually. Alternatively, test via the Canvas App scrNoviPredmet screen once the app is built.

After each test, verify:
1. Flow run history.
2. Svi predmeti list — presence or absence of new item.
3. Dokumenti library — presence or absence of new file(s).
4. RezervisaniBrojevi — presence or absence of reserved number item.
5. App Config — counter value (for generated number tests).
6. AuditLog — all expected entries with correct severity and status.

---

## TC-001: Happy path — new number, main file only, no attachments

**Purpose:** Full success path with auto-generated number.

**Input:**
```json
{
  "documentType": "<valid type from App Config>",
  "useReservedNumber": false,
  "reservedNumberId": 0,
  "filingDate": "",
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-001",
  "metadataJson": "{\"subject\":\"Test dokument TC-001\",\"description\":\"Opis\",\"partnerNaziv\":\"Test Partner\",\"partnerId\":null,\"partnerPIBSnapshot\":\"\",\"partnerMestoSnapshot\":\"\",\"partnerAdresaSnapshot\":\"\",\"customFields\":{}}",
  "mainFileJson": "{\"fileName\":\"test-dokument.pdf\",\"fileContentBase64\":\"<base64_of_small_pdf>\"}",
  "attachmentsJson": "[]"
}
```

**Expected response:**
```json
{
  "success": true,
  "message": "Dokument je uspešno zaveden.",
  "itemId": <positive integer>,
  "delovodniBroj": "<counter>/2026",
  "correlationId": "test-correlation-001",
  "errorCode": ""
}
```

**Verify:**
- Svi predmeti item created: Title = `Test dokument TC-001`, DelovodniBroj = returned value, Stanje = `Zavedeno`.
- Dokumenti file created with system name containing SafeDelovodniBroj + guid + original name.
- File IsPrilog = false in metadata.
- App Config counter incremented by 1.
- AuditLog: Started, Success entries for CreateDocument.

**Pass criteria:** Flow green. All artifacts verified. Counter incremented once.

---

## TC-002: Happy path — reserved number, with filing date and attachment

**Purpose:** Full success path using a reserved number and one attachment.

**Setup:** A valid RezervisaniBrojevi item exists (ID = `<testReservedId>`, year = active year).

**Input:**
```json
{
  "documentType": "<valid type>",
  "useReservedNumber": true,
  "reservedNumberId": <testReservedId>,
  "filingDate": "2026-01-20",
  "initiatorEmail": "test.user@organizacija.rs",
  "correlationId": "test-correlation-002",
  "metadataJson": "{\"subject\":\"Test dokument TC-002\",\"description\":\"\",\"partnerNaziv\":\"\",\"partnerId\":null,\"partnerPIBSnapshot\":\"\",\"partnerMestoSnapshot\":\"\",\"partnerAdresaSnapshot\":\"\",\"customFields\":{}}",
  "mainFileJson": "{\"fileName\":\"ugovor.pdf\",\"fileContentBase64\":\"<base64_of_small_pdf>\"}",
  "attachmentsJson": "[{\"fileName\":\"prilog.pdf\",\"fileContentBase64\":\"<base64_of_small_pdf>\"}]"
}
```

**Expected:**
- success = true.
- RezervisaniBrojevi item deleted.
- Two files in Dokumenti: main file (IsPrilog=false), attachment (IsPrilog=true, _PRILOG_ in name).
- FilingDate on Svi predmeti item = `2026-01-20`.
- AuditLog: UseReservedNumber/Success entry (deletion) present.

**Pass criteria:** Flow green. Reserved number item gone. Two files with correct metadata.

---

## TC-003: Validation failure — missing documentType

**Purpose:** Verify VALIDATION_FAILED when documentType is empty.

**Input:** TC-001 but `"documentType": ""`.

**Expected:**
```json
{ "success": false, "errorCode": "VALIDATION_FAILED", "message": "Tip dokumenta je obavezan." }
```

No Svi predmeti item. No file. Counter not incremented. Reserved number untouched.

**Pass criteria:** Structured failure. Clean state confirmed.

---

## TC-004: Validation failure — missing mainFile

**Purpose:** Verify VALIDATION_FAILED when mainFileJson has no fileContentBase64.

**Input:** TC-001 but `"mainFileJson": "{\"fileName\":\"test.pdf\",\"fileContentBase64\":\"\"}"`.

**Expected:** `{ "success": false, "errorCode": "VALIDATION_FAILED", "message": "Sadržaj fajla je obavezan." }`

**Pass criteria:** Clean state. No artifacts created.

---

## TC-005: Invalid document type

**Purpose:** Verify INVALID_DOCUMENT_TYPE when documentType is not in App Config.

**Input:** TC-001 but `"documentType": "NEPOSTOJECI_TIP"`.

**Expected:** `{ "success": false, "errorCode": "INVALID_DOCUMENT_TYPE" }`

**Pass criteria:** Clean state. No number generated.

---

## TC-006: Reserved number not found

**Purpose:** Verify RESERVED_NUMBER_INVALID when reservedNumberId does not exist.

**Input:** TC-002 but `"reservedNumberId": 99999` (non-existent ID).

**Expected:** `{ "success": false, "errorCode": "RESERVED_NUMBER_INVALID" }`

**Pass criteria:** Clean state. No artifacts created.

---

## TC-007: Reserved number from wrong year

**Purpose:** Verify RESERVED_NUMBER_INVALID (RESERVED_NUMBER_YEAR_MISMATCH from child flow) when reserved number is from last year.

**Setup:** A RezervisaniBrojevi item exists with year = prior year.

**Input:** TC-002 but reservedNumberId = prior year item. filingDate = `2026-01-20`.

**Expected:** Failure. Reserved number untouched.

**Pass criteria:** RESERVED_NUMBER_INVALID returned. Item not deleted.

---

## TC-008: Multiple attachments

**Purpose:** Verify all attachments are uploaded and correctly linked.

**Input:** TC-001 but with 3 attachments in attachmentsJson.

**Expected:**
- 1 main file + 3 attachments in Dokumenti (4 files total).
- All 4 files have same DelovodniBroj and SviPredmetiId.
- Attachments: IsPrilog=true, system name contains `_PRILOG_`.
- Main file: IsPrilog=false.

**Pass criteria:** 4 files in Dokumenti. All metadata correct.

---

## TC-009: One attachment fails, others succeed (simulated)

**Purpose:** Verify that one failed attachment does not fail the entire flow.

**Simulation:** Pass one attachment with invalid Base64 content (e.g. `"fileContentBase64": "INVALID"`).

**Expected:**
- Main file created successfully.
- Failed attachment logged as Warning.
- Flow returns success (document is created).
- AuditLog: Warning for the failed attachment.

**Pass criteria:** success = true. Main file exists. AuditLog Warning for attachment.

---

## TC-010: File name with special characters

**Purpose:** Verify that file names with `/`, `:`, `?`, and spaces are sanitized and do not cause upload failures.

**Input:** TC-001 but `"mainFileJson": "{\"fileName\":\"ugovor: 2026/01 <test>.pdf\",\"fileContentBase64\":\"<base64>\"}"`.

**Expected:**
- File created in Dokumenti (no SharePoint error on special characters).
- System file name does not contain `/`, `:`, `<`, or `>`.
- OriginalFileName metadata = `ugovor: 2026/01 <test>.pdf` (original preserved).

**Pass criteria:** Upload succeeds. System name is clean. Original name in metadata.

---

## TC-011: Permission failure — non-fatal

**Purpose:** Verify that permission failure does not prevent document creation.

**Simulation:** Temporarily break CF_DocCentralV3_AssignPermissions (e.g. remove service account permissions from Svi predmeti so break inheritance fails).

**Expected:**
```json
{
  "success": true,
  "message": "Dokument je zaveden ali dodela prava nije uspela. Kontaktirajte administratora.",
  "itemId": <positive integer>,
  "delovodniBroj": "<number>/2026",
  "errorCode": "PERMISSION_ASSIGNMENT_FAILED"
}
```

**Restore:** Reinstate service account permissions after test.

**Pass criteria:** success = true. Document item and file exist. Warning message visible.

---

## TC-012: Svi predmeti Create item fails (compensation)

**Purpose:** Verify compensation delete is attempted when Svi predmeti creation fails after number generation.

**Simulation:** Temporarily rename or lock the Svi predmeti list (or remove service account write permission) so Create item returns an error.

**Expected:**
- Number was generated (check App Config counter incremented).
- No Svi predmeti item exists.
- AuditLog: Error entry with the generated delovodniBroj recorded in the error message for reconciliation.
- Flow returns failure: `{ "success": false, "errorCode": "CREATE_ITEM_FAILED" }`.

**Restore:** Reinstate list access after test.

**Pass criteria:** Structured failure. No orphan Svi predmeti item. Counter incremented (gap is expected and acceptable).

---

## TC-013: Concurrent submissions — no duplicate numbers

**Purpose:** Verify two simultaneous calls each get a unique delovodniBroj.

**Method:** Trigger two simultaneous calls from two parallel branches in a test flow.

**Expected:**
- Two Svi predmeti items with distinct delovodniBroj values.
- App Config counter incremented by exactly 2.
- No duplicate numbers.

**Pass criteria:** Two unique numbers. Counter at original + 2. No REGISTRY_NUMBER_GENERATION_FAILED errors.

---

## TC-014: Filing date set correctly

**Purpose:** Verify FilingDate is set to today for new numbers, and to user-selected date for reserved numbers.

**Method:**
1. Run TC-001 (new number). Check FilingDate on Svi predmeti item = today's UTC date.
2. Run TC-002 with filingDate = `2026-01-15`. Check FilingDate = `2026-01-15`.

**Pass criteria:** Correct FilingDate in both cases.

---

## TC-015: Empty correlationId — auto-generation

**Purpose:** Verify GUID auto-generation.

**Input:** TC-001 with `"correlationId": ""`.

**Expected:** Response correlationId is a valid 36-character GUID. All AuditLog entries share it.

**Pass criteria:** Valid GUID in response and AuditLog.

---

## Post-test cleanup

1. Delete test Svi predmeti items created during testing.
2. Delete test files from Dokumenti library.
3. Restore App Config counter if modified.
4. Restore service account permissions if changed for TC-011, TC-012.
5. Restore Svi predmeti access if changed for TC-012.
6. Leave any RezervisaniBrojevi test items that still exist for future reserved number testing.
7. Confirm flow is inside DocCentralV3 solution.
