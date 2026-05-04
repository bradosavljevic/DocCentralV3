# Test cases: CF_DocCentralV3_ProcessApprovalResponse

## Prerequisites

- A Svi predmeti item has been put through CF_DocCentralV3_SendForApproval (Stanje = `U odobravanju`).
  Note: itemId, delovodniBroj, step 1 assignee, KoraciOdobravanjaJson.
- CF_DocCentralV3_LogEvent, CF_DocCentralV3_AssignPermissions published.
- Two user accounts available: step 1 assignee, and an unauthorized user.
- For group tests: a test Entra group with 2+ known member emails.

---

## TC-001: Single step, user approves — ProcessComplete

**Setup:** Item in `U odobravanju`, KoraciOdobravanjaJson = [{stepNumber:1, status:Pending}], UkupnoKoraka = 1.

**Input:**
```json
{ "documentItemId": <id>, "outcome": "Approved", "comments": "", "responderEmail": "step1@organizacija.rs", "responderDisplayName": "Step1 User", "correlationId": "tc-001" }
```

**Expected:**
- success = true. nextAction = `ProcessComplete`.
- Svi predmeti: Stanje = `Odobreno`. TrenutniOdobravalacEmail = `''`. TrenutnaGrupaOdobravanjaId = `''`.
- KoraciOdobravanjaJson: step 1 status = `Approved`, resolvedBy = `step1@organizacija.rs`, resolvedAt set.
- IstorijaOdobravanjaJson: 1 entry, outcome = `Approved`.
- Notification email sent to InicijatorEmail (process complete).
- AuditLog: ProcessApprovalResponse / Info / Started + Success.

**Pass criteria:** Flow green. All fields correct. Initiator notified.

---

## TC-002: Step 1 of 2 approved — NextStepActivated

**Setup:** Item in `U odobravanju`, 2 steps: step 1 = User, step 2 = User. TrenutniKorak = 1.

**Input:** Same as TC-001 but with 2-step item.

**Expected:**
- success = true. nextAction = `NextStepActivated`.
- Stanje = `U odobravanju` (unchanged).
- TrenutniKorakOdobravanja = 2. TrenutniOdobravalacEmail = step2 email.
- KoraciOdobravanjaJson: step 1 Approved, step 2 still Pending.
- Notification email sent to step 2 assignee.
- Initiator NOT notified (process not complete).

**Pass criteria:** Step 2 activated. Step 1 resolved. Correct notification.

---

## TC-003: Rejection — DocumentRejected, remaining steps Skipped

**Setup:** 2-step item at step 1 (TrenutniKorak = 1).

**Input:** `"outcome": "Rejected", "comments": "Nedostaje prilog."`

**Expected:**
- success = true. nextAction = `DocumentRejected`.
- Stanje = `Odbijeno`. TrenutniOdobravalacEmail = `''`.
- KoraciOdobravanjaJson: step 1 = Rejected, step 2 = Skipped.
- IstorijaOdobravanjaJson: rejection entry appended.
- Initiator notified by email.

**Pass criteria:** Both steps resolved (Rejected/Skipped). Initiator email sent.

---

## TC-004: Group step — first member approves

**Setup:** Item in `U odobravanju`, step 1 = Group type (varGrupaId set). TrenutniOdobravalacEmail = `''`.

**Input:** `"responderEmail": "<member1@org.rs>"` (confirmed group member).

**Expected:**
- Authorization passes (member found in group).
- success = true. Step resolved.

**Pass criteria:** Flow green. Group member authorized.

---

## TC-005: Group step — second member tries after first resolved (STEP_ALREADY_RESOLVED)

**Method:**
1. Run TC-004 (first member approves).
2. Before the Svi predmeti PATCH is committed by TC-004, attempt TC-004 simultaneously with `"responderEmail": "<member2@org.rs>"`. (Simulate by running TC-004 again after TC-004 completes — the step is now Approved.)

**Expected:**
- For the second call: `{ "success": false, "errorCode": "STEP_ALREADY_RESOLVED" }`.

**Pass criteria:** STEP_ALREADY_RESOLVED returned. No double-resolution.

---

## TC-006: RESPONDER_NOT_AUTHORIZED — wrong user

**Input:** `"responderEmail": "wrong.user@organizacija.rs"` (not the current step assignee and not in group).

**Expected:** `{ "success": false, "errorCode": "RESPONDER_NOT_AUTHORIZED" }`. Item unchanged.

---

## TC-007: NOT_IN_APPROVAL

**Setup:** Item with Stanje = `Zavedeno` (not in approval).

**Expected:** `{ "success": false, "errorCode": "NOT_IN_APPROVAL" }`.

---

## TC-008: APPROVAL_STATE_INVALID — KoraciJson empty

**Setup:** Item with Stanje = `U odobravanju` but KoraciOdobravanjaJson = null or `[]`.
(Simulate by manually blanking the column — for testing only.)

**Expected:** `{ "success": false, "errorCode": "APPROVAL_STATE_INVALID" }`.

---

## TC-009: INVALID_OUTCOME

**Input:** `"outcome": "Maybe"`.

**Expected:** `{ "success": false, "errorCode": "INVALID_OUTCOME" }`. Validated before item read.

---

## TC-010: Comments preserved in both JSON fields

**Input:** TC-001 but with `"comments": "Odobravam uz napomenu."`.

**Expected:**
- KoraciOdobravanjaJson step 1: `comments = "Odobravam uz napomenu."`.
- IstorijaOdobravanjaJson entry: `comments = "Odobravam uz napomenu."`.

**Pass criteria:** Comment string intact in both fields.

---

## TC-011: IstorijaJson accumulates across multiple rounds

**Method:**
1. Run SendForApproval (round 1).
2. Run ProcessApprovalResponse → Reject (adds 1 history entry).
3. Run SendForApproval again (resubmission — adds RoundReset marker).
4. Run ProcessApprovalResponse → Approve (adds another history entry, round = 2).
5. Check IstorijaOdobravanjaJson.

**Expected:** 3 entries: rejection (round 1), RoundReset, approval (round 2).

**Pass criteria:** History accumulates. Round numbers correct. Not replaced.

---

## TC-012: 412 conflict — detect step already resolved

**Method:**
1. Manually set the step status in KoraciOdobravanjaJson to `Approved` (simulating another user resolved it).
2. Call ProcessApprovalResponse with a valid responder.
3. The PATCH will get a 412 (ETag mismatch from the manual edit).
4. Re-read detects step status is no longer Pending.

**Expected:** `{ "success": false, "errorCode": "STEP_ALREADY_RESOLVED" }` returned after retry.

**Pass criteria:** 412 handled. STEP_ALREADY_RESOLVED returned. No data corruption.

---

## TC-013: Email failure — non-fatal

**Simulation:** Disconnect Outlook or use an invalid recipient email.

**Expected:** success = true. nextAction correct. AuditLog Warning for email failure.

---

## TC-014: Permission grant — non-fatal on failure

For NextStepActivated: simulate AssignPermissions failure.

**Expected:** success = true. Warning logged. Next step fields updated correctly.

---

## Post-test cleanup

1. Reset test Svi predmeti item Stanje and all approval fields for re-use.
2. Delete test AuditLog entries.
3. Confirm flow is inside DocCentralV3 solution.
