# Test cases: CF_DocCentralV3_ExportAppConfig

## Prerequisites

- App Config list populated with at least one item.
- Exports library exists and flow service account has Write access.
- CF_DocCentralV3_LogEvent published.
- Admin user account available for testing.

---

## TC-001: Normal export — App Config with items

**Setup:** App Config has N items (e.g. 20).

**Input:**
```json
{ "initiatorEmail": "admin@organizacija.rs", "correlationId": "tc-001" }
```

**Expected:**
- success = true.
- fileUrl and fileName populated.
- CSV file exists in Exports library root with name `AppConfig_Export_<timestamp>.csv`.
- File opens correctly in Excel — Serbian characters render correctly (UTF-8 BOM present).
- Row count in CSV = N + 1 (header row).
- Column order matches agreed export schema.
- AuditLog: Started + Success entries.

**Pass criteria:** File created. All N rows present. Excel opens without encoding errors.

---

## TC-002: Empty App Config list

**Setup:** App Config list temporarily emptied (remove all items for test — restore after).

**Expected:**
- success = true.
- CSV file created with header row only.
- No data rows.
- fileUrl returned.

**Pass criteria:** Flow does not error on empty result. Empty CSV created successfully.

---

## TC-003: Values containing commas — CSV escaping

**Setup:** Add an App Config item where Value = `"Rate, per day"` (contains a comma).

**Expected:**
- CSV row wraps the Value field in double-quotes: `"Rate, per day"`.
- Opening the file in Excel shows the value as `Rate, per day` in a single cell (not split across cells).

**Pass criteria:** Comma in value does not break CSV structure.

---

## TC-004: Values containing double-quotes — CSV escaping

**Setup:** Add an App Config item where Value = `He said "yes"`.

**Expected:**
- CSV row escapes internal double-quotes as `""`.
- Result in CSV: `"He said ""yes"""`
- Excel shows: `He said "yes"`.

**Pass criteria:** Double-quote in value does not break CSV parsing.

---

## TC-005: Values containing Serbian characters

**Setup:** App Config item with Value = `Naziv organizacije: Фирма д.о.о.` or similar Serbian text.

**Expected:**
- CSV file opens in Excel with characters rendered correctly (č, ć, š, ž, đ visible — not as `?` or garbage).
- UTF-8 BOM present in file (verify: open in Notepad++ and check encoding shows UTF-8 BOM).

**Pass criteria:** Encoding correct. No character corruption.

---

## TC-006: Null or empty field in App Config item

**Setup:** App Config item with Description = null/empty.

**Expected:**
- CSV row has empty string in the Description column position (consecutive commas or empty quoted string).
- File parses correctly in Excel — no row shift.

**Pass criteria:** Null fields handled gracefully.

---

## TC-007: Multiple exports — distinct filenames

**Action:** Run export twice within the same minute.

**Expected:**
- Two distinct files in Exports library (timestamps differ by seconds).
- First file not overwritten.

**Pass criteria:** File naming uses HHmmss — distinct even within the same minute if run quickly. (If both run within the same second, filenames may collide — acceptable edge case for a manual admin operation.)

---

## TC-008: Exports library full or Write access denied

**Method:** Revoke service account Write access to the Exports library. Run export.

**Expected:**
- success = false. errorCode = `EXPORT_FAILED`.
- AuditLog: Error entry with error message.
- No partial file created.

**Pass criteria:** Catch scope handles SharePoint Create file failure. Error returned cleanly.

---

## TC-009: Canvas App fileUrl opens correct file

**After TC-001:** Use the returned fileUrl in a browser (or via Canvas App Launch()).

**Expected:**
- Browser downloads or displays the CSV file.
- File content matches App Config data.

**Pass criteria:** fileUrl is valid and accessible by the admin user.

---

## TC-010: correlationId auto-generated when empty

**Input:** `"correlationId": ""`

**Expected:**
- Response correlationId is a valid GUID.
- AuditLog entries reference the same GUID.

---

## Post-test cleanup

1. Delete test CSV files from Exports library.
2. Remove any test App Config items added for escape testing.
3. Delete AuditLog entries.
4. Confirm flow is inside DocCentralV3 solution.
