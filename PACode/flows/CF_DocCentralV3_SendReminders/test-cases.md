# Test cases: CF_DocCentralV3_SendReminders

## Prerequisites

- Podsetnici list exists with all required columns (ReminderDate, RecipientEmails, IsSent, CreatedByEmail).
- IsSent column confirmed as Yes/No type.
- CF_DocCentralV3_LogEvent published.
- Valid Outlook connection reference for email sending.
- Test email addresses available (use real addresses or a distribution list for testing).

**Testing note:** Scheduled flows cannot be triggered from Canvas App. Use Power Automate "Run" button manually for testing. Or set recurrence to a near-future time, let it fire, then verify.

---

## TC-001: One reminder due today, one recipient

**Setup:** Podsetnici item with ReminderDate = today, RecipientEmails = `test.user@organizacija.rs`, IsSent = false.

**Action:** Run flow manually in Power Automate.

**Expected:**
- Email received at `test.user@organizacija.rs` with subject `DocCentral podsetnik: <Title>`.
- Podsetnici item: IsSent = true.
- AuditLog: Started + per-reminder Success + overall Success.
- varSentCount = 1. varFailedCount = 0.

**Pass criteria:** Email delivered. IsSent updated. Audit log correct.

---

## TC-002: One reminder, multiple recipients

**Setup:** Podsetnici item with ReminderDate = today, RecipientEmails = `user1@org.rs;user2@org.rs` (or two People entries).

**Expected:**
- Both users receive the email.
- IsSent = true on the Podsetnici item.
- AuditLog: 1 per-reminder success log (not one per recipient).

**Pass criteria:** Both emails received. Single reminder counted as 1 success.

---

## TC-003: No reminders due today

**Setup:** Podsetnici list empty, or all items have ReminderDate ≠ today or IsSent = true.

**Expected:**
- No emails sent.
- AuditLog: Started + Info (no reminders due). No Error or Warning.
- varSentCount = 0. varFailedCount = 0.

**Pass criteria:** Flow completes cleanly. No email activity.

---

## TC-004: Reminder has IsSent = true — skipped

**Setup:** Podsetnici item with ReminderDate = today, IsSent = true.

**Expected:**
- Item filtered out by OData query — not processed.
- No email sent.

**Pass criteria:** IsSent filter works. Already-sent reminder not reprocessed.

---

## TC-005: Flow runs twice same day — idempotency

**Method:** Run the flow twice manually on the same day. (First run processes reminders. Second run should find IsSent = true and skip.)

**Expected:**
- First run: emails sent, IsSent updated.
- Second run: no items returned from Get_DueReminders. AuditLog: no reminders due.

**Pass criteria:** No duplicate emails on second run.

---

## TC-006: Reminder has no recipients — warning logged, IsSent marked

**Setup:** Podsetnici item with ReminderDate = today, RecipientEmails = empty or `''`, IsSent = false.

**Expected:**
- AuditLog Warning: podsetnik has no valid recipients.
- IsSent = true (item marked to prevent repeated warning on future runs).
- varSentCount not incremented (no email sent). varFailedCount incremented.

**Pass criteria:** Warning logged. No exception. IsSent set to prevent repeated processing.

---

## TC-007: Email delivery fails for one recipient in multi-recipient reminder

**Method:** Add an invalid/non-existent email address as one of the recipients (e.g. `nonexistent@fake.domain`). Include a valid recipient as well.

**Expected:**
- Valid recipient: email delivered.
- Invalid recipient: Outlook action fails. AuditLog Warning per failed recipient.
- IsSent = true (marked regardless of email failure — prevents reprocessing).
- Flow continues. varFailedCount incremented.

**Pass criteria:** Partial success handled. IsSent still set. Other recipients unaffected.

---

## TC-008: IsSent update fails — warning logged, no flow failure

**Method:** Temporarily revoke flow's write access to Podsetnici list (or simulate by renaming the IsSent column after flow reads it).

**Expected:**
- Update item action fails.
- AuditLog Warning: IsSent update failed; reminder may be resent.
- Flow does not throw unhandled error. Loop continues to next reminder.

**Pass criteria:** IsSent failure is Warning-only. Per-reminder failure does not crash loop.

---

## TC-009: Multiple reminders due same day — all processed

**Setup:** 3 Podsetnici items with ReminderDate = today, IsSent = false.

**Expected:**
- All 3 processed in sequence.
- All 3 get emails. All 3 have IsSent = true after run.
- AuditLog: 3 per-reminder success entries + overall success.
- varSentCount = 3.

**Pass criteria:** Loop processes all items. No short-circuit on success or failure.

---

## TC-010: ReminderDate is tomorrow — not processed

**Setup:** Item with ReminderDate = tomorrow.

**Expected:**
- Not returned by OData filter.
- No email. No log entry for this item.

**Pass criteria:** Date filter is exact (not >=).

---

## TC-011: AuditLog severity on partial failure

**Setup:** 2 reminders. Reminder A: valid. Reminder B: no recipients.

**Expected:**
- Overall log: severity = Warning.
- varSentCount = 1. varFailedCount = 1.

**Pass criteria:** Warning severity when any reminder fails.

---

## Post-test cleanup

1. Reset all test Podsetnici items: set IsSent = false (or delete items).
2. Delete AuditLog entries created during testing.
3. Confirm flow is inside DocCentralV3 solution.
4. Confirm Concurrency Control = 1 in flow settings (verify this was not reset).
