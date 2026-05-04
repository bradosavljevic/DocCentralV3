# Test cases: CF_DocCentralV3_GenerateArchiveBookPdf

## Prerequisites

- At least one Svi predmeti item with Stanje = `Arhivirano` and the correct year.
- Exports library exists and flow service account has Write access.
- CF_DocCentralV3_LogEvent published.
- App Config has organization name entry (UNKNOWN key — or verify fallback to placeholder).

---

## TC-001: Year with archived documents — HTML file generated

**Setup:** 3 Svi predmeti items with Stanje = `Arhivirano` for year 2026.

**Input:**
```json
{ "year": 2026, "initiatorEmail": "admin@organizacija.rs", "correlationId": "tc-001" }
```

**Expected:**
- success = true. fileUrl and fileName populated.
- HTML file `ArhivskaKnjiga_2026_<timestamp>.html` exists in Exports library root.
- File has 3 data rows + header row.
- Organization name in file header (or placeholder if App Config key not configured).
- Total document count shown correctly.
- AuditLog: Started + Success entries.

**Pass criteria:** Flow green. File accessible. Rows correct.

---

## TC-002: Year with no archived documents — empty HTML file

**Setup:** No Svi predmeti items with Stanje = `Arhivirano` for year 2025.

**Input:** `"year": 2025`

**Expected:**
- success = true.
- HTML file created with header and one empty-state row: `Nema arhiviranih dokumenata za godinu 2025.`
- totalCount = 0. 0 data rows in table.

**Pass criteria:** Flow does not error on empty result. Empty-state message correct.

---

## TC-003: Serbian characters in document fields

**Setup:** Svi predmeti item with Subject = `Ugovor o saradnji — žalba`. DelovodniBroj with standard Serbian characters.

**Expected:**
- HTML file opens in browser — all Serbian characters render correctly (no `?` or garbled text).
- UTF-8 BOM present (verify in text editor: file starts with BOM marker, Notepad++ shows UTF-8 BOM).

**Pass criteria:** Encoding correct. No character corruption.

---

## TC-004: HTML injection — values containing HTML characters

**Setup:** Svi predmeti item with Subject = `Report <2026> & "Annual"`.

**Expected:**
- HTML file shows: `Report &lt;2026&gt; &amp; &quot;Annual&quot;` in source.
- Browser renders: `Report <2026> & "Annual"` (correct display).
- HTML structure not broken (no extra tags injected).

**Pass criteria:** HTML escaping applied correctly to all text fields.

---

## TC-005: Date formatting — Arhivirano and FilingDate

**Setup:** Archived document with FilingDate = `2026-03-15` and Arhivirano = `2026-04-01`.

**Expected:**
- HTML shows: Datum zavođenja = `15.03.2026`. Datum arhiviranja = `01.04.2026`.

**Pass criteria:** `dd.MM.yyyy` format applied correctly to both date fields.

---

## TC-006: Null/empty date field

**Setup:** Archived document with FilingDate = null (not filled in).

**Expected:**
- HTML shows empty string in Datum zavođenja cell (not `null` or `0001-01-01`).
- No flow error.

**Pass criteria:** Null-safe date formatting works. Empty cell rendered cleanly.

---

## TC-007: Organization name from App Config

**Setup:** App Config has an organization name entry (UNKNOWN key = e.g. `OrgName` with value `Firma d.o.o.`).

**Expected:**
- HTML header shows: `FIRMA D.O.O.` (or as stored in App Config).
- Not the placeholder `[Naziv organizacije]`.

**Pass criteria:** Org name read from App Config and inserted correctly.

---

## TC-008: Organization name not in App Config — placeholder used

**Setup:** App Config org name key does not exist or returns empty.

**Expected:**
- HTML header shows: `[Naziv organizacije]`.
- Flow does not error. success = true.

**Pass criteria:** Graceful fallback to placeholder.

---

## TC-009: Print layout — A4 landscape

**Action:** Open generated HTML file in Chrome. Press Ctrl+P.

**Expected:**
- Print preview shows A4 landscape orientation (set by `@page` CSS rule).
- Table header repeats on every printed page (`display: table-header-group`).
- No row split across page boundaries (`page-break-inside: avoid`).

**Pass criteria:** HTML is print-ready. Print preview looks correct.

---

## TC-010: Multiple rows — correct row numbering

**Setup:** 5 archived documents for the year.

**Expected:**
- Rows numbered 1, 2, 3, 4, 5 in sequence.
- Row numbers match the sorted order (by DelovodniBroj or FilingDate ascending).

**Pass criteria:** Row numbering sequential and matches sort order.

---

## TC-011: Document count > 5000 — pagination (if implemented)

**Setup:** Svi predmeti has more than 5000 archived documents for the year.
(If pagination was not implemented, document this as a known limitation and skip this test case.)

**Expected:**
- All documents appear in the HTML file.
- totalCount in the file header matches actual item count.
- Row count = totalCount.

**Pass criteria:** All pages retrieved. No documents missing.

---

## TC-012: Exports library Write access denied

**Method:** Revoke service account Write access to Exports library. Run flow.

**Expected:**
- success = false. errorCode = `ARCHIVE_BOOK_GENERATION_FAILED`.
- AuditLog: Error entry.
- No partial HTML file.

**Pass criteria:** Catch scope handles Create file failure cleanly.

---

## TC-013: Previous year (closed year) — no restriction

**Input:** `"year": 2024` (a previously closed year).

**Expected:**
- Flow runs identically — no restriction based on year value.
- Retrieves all archived documents for 2024 regardless of year status.

**Pass criteria:** Year value is not validated against App Config active year. Any year works.

---

## TC-014: fileUrl opens the correct file

**After TC-001:** Use the returned fileUrl in a browser or via Canvas App `Launch()`.

**Expected:**
- Browser opens the HTML archive book.
- Content matches the archived documents for year 2026.

**Pass criteria:** fileUrl is valid and accessible to the admin user.

---

## Post-test cleanup

1. Delete test HTML files from Exports library.
2. Delete AuditLog entries from testing.
3. Confirm flow is inside DocCentralV3 solution.
