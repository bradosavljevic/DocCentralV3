# Test cases: CF_DocCentralV3_SendForApproval

## Prerequisites

- A Svi predmeti item exists with Stanje = `Zavedeno`, ID noted as `<testItemId>`.
- App Config has ProcesConfig for the test document type.
- CF_DocCentralV3_AssignPermissions published.
- Office365Groups connection reference connected (for group step tests).
- Outlook connection reference connected.

---

## TC-001: Single user step — happy path

**Setup:** ProcesConfig for `documentType` = 1 step, User type, step1 assigneeEmail = `odobravalac@organizacija.rs`.

**Input:**
```json
{ "documentItemId": <testItemId>, "delovodniBroj": "17/2026", "documentType": "Ulazni", "initiatorEmail": "korisnik@organizacija.rs", "initiatorDisplayName": "Korisnik", "correlationId": "tc-001" }
```

**Expected:**
- success = true. totalSteps = 1. firstApprover = `odobravalac@organizacija.rs`.
- Svi predmeti item: Stanje = `U odobravanju`, TrenutniKorakOdobravanja = 1, TrenutniOdobravalacEmail = `odobravalac@organizacija.rs`, TrenutnaGrupaOdobravanjaId = `''`.
- KoraciOdobravanjaJson: 1 step, status = `Pending`.
- IstorijaOdobravanjaJson = `[]`.
- Notification email sent to `odobravalac@organizacija.rs`.
- AuditLog: SendForApproval / Info / Started + Success.

**Pass criteria:** Flow green. All fields correct. Email delivered.

---

## TC-002: Two sequential steps — both in KoraciOdobravanjaJson

**Setup:** ProcesConfig = 2 steps. Step 1: User. Step 2: Group.

**Expected:**
- KoraciOdobravanjaJson contains both steps, both Pending.
- TrenutniKorakOdobravanja = 1. Only step 1 fields set on item.
- Notification sent only to step 1 assignee.

**Pass criteria:** Both steps in JSON. Only step 1 activated on item.

---

## TC-003: Group step — all group members notified

**Setup:** ProcesConfig = 1 step, Group type, assigneeGroupId = `<testGroupId>` (group with 2 known members).

**Expected:**
- TrenutnaGrupaOdobravanjaId = `<testGroupId>`. TrenutniOdobravalacEmail = `''`.
- Notification emails sent to both group members.

**Pass criteria:** Two emails sent. Group ID on item. User email empty.

---

## TC-004: ALREADY_IN_APPROVAL

**Setup:** Svi predmeti item with Stanje = `U odobravanju`.

**Expected:** `{ "success": false, "errorCode": "ALREADY_IN_APPROVAL" }`. No fields changed.

**Pass criteria:** Structured failure. Item unchanged.

---

## TC-005: DOCUMENT_ARCHIVED

**Setup:** Stanje = `Arhivirano`.

**Expected:** `{ "success": false, "errorCode": "DOCUMENT_ARCHIVED" }`.

---

## TC-006: NO_PROCESS_CONFIG

**Input:** `"documentType": "NEPOSTOJECI_TIP"` (not in App Config).

**Expected:** `{ "success": false, "errorCode": "NO_PROCESS_CONFIG" }`. Item Stanje unchanged.

---

## TC-007: Resubmission after rejection

**Setup:** Svi predmeti item with Stanje = `Odbijeno`, IstorijaOdobravanjaJson = existing entries from previous round.

**Expected:**
- success = true.
- IstorijaOdobravanjaJson: existing entries + RoundReset marker appended. NOT replaced.
- KoraciOdobravanjaJson: fresh array with all steps Pending.
- Stanje = `U odobravanju`.
- varCurrentRound = 2.

**Pass criteria:** History preserved. KoraciJson reset. RoundReset marker present.

---

## TC-008: Email failure is non-fatal

**Simulation:** Configure invalid recipient email or disconnect Outlook connection temporarily.

**Expected:** success = true. AuditLog Warning entry for email failure.

**Pass criteria:** Flow returns success. Warning in AuditLog.

---

## TC-009: Permission assignment failure is non-fatal

**Simulation:** Break AssignPermissions (e.g. remove service account permissions).

**Expected:** success = true. AuditLog Warning for permission failure. Stanje updated correctly.

---

## TC-010: 412 conflict on PATCH — retry succeeds

**Simulation:** Difficult to simulate directly. Use a concurrent test:
Two simultaneous calls to SendForApproval for the same item.

**Expected:** One call succeeds (acquires the PATCH). The other returns ALREADY_IN_APPROVAL (detects Stanje = U odobravanju after re-read on 412).

**Pass criteria:** Exactly one success. One ALREADY_IN_APPROVAL. No data corruption.

---

## TC-011: Empty correlationId — auto-generation

**Input:** `"correlationId": ""`.

**Expected:** Response correlationId is a valid GUID. All AuditLog entries share it.

---

## Post-test cleanup

1. Reset Svi predmeti item Stanje back to `Zavedeno` for re-use.
2. Clear KoraciOdobravanjaJson and IstorijaOdobravanjaJson if needed.
3. Delete test AuditLog entries.
4. Confirm flow is inside DocCentralV3 solution.
