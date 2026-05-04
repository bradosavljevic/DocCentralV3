# Rollback and compensation: CF_DocCentralV3_CreateDocument

## Why rollback is partial in Power Automate

Power Automate has no built-in transaction mechanism. Once a SharePoint item or file is created,
the creation cannot be automatically undone if a later step fails — there is no ROLLBACK statement.

DocCentralV3 handles this with a **compensation** approach:
- Before any write, the flow is at a known clean state.
- After each write, a failure at the next step is handled by logging the partial state and returning a structured error.
- Where feasible, a cleanup step deletes the partially created artifact before returning the error.
- Where cleanup itself could fail, the failure is logged as a Critical entry requiring manual administrator action.

---

## State machine at each flow step

### Clean state (before Step 5d — number acquisition)

| Artifact | State |
|---|---|
| Svi predmeti item | Does not exist |
| Dokumenti file | Does not exist |
| Registry counter (App Config) | Not incremented |
| Reserved number (RezervisaniBrojevi) | Untouched |

**Failure handling:** Any failure before or during number acquisition returns a clean failure.
No artifacts exist. The user may retry immediately.

---

### After Step 5d — number acquired (before Svi predmeti creation)

| Artifact | State |
|---|---|
| Svi predmeti item | Does not exist |
| Dokumenti file | Does not exist |
| Registry counter | **Incremented** (if generated number) |
| Reserved number | **Validated but not deleted** |

**Failure handling at this point (e.g. Svi predmeti Create item fails):**

The registry number has been "consumed" at the App Config level.
The Svi predmeti item does not exist.

**Compensation action:**
1. Log error with the generated delovodniBroj explicitly in the error message and detailsJson:
   ```
   "Dokument nije kreiran. Generisani delovodni broj: 42/2026 nije dodeljen nijednom dokumentu. Potrebna ručna provera."
   ```
2. Severity: Error. The delovodniBroj must appear in the AuditLog for administrator reconciliation.
3. Do NOT attempt to decrement the App Config counter — this could cause a second race condition.
4. Return CREATE_ITEM_FAILED to the Canvas App.
5. The user must contact an administrator to determine whether the number is available.

For reserved number path: the reserved number item still exists in RezervisaniBrojevi.
Do NOT attempt to "un-validate" it. It remains available for the user's next attempt.

---

### After Step 5f — Svi predmeti item created (before main file creation)

| Artifact | State |
|---|---|
| Svi predmeti item | **Exists** (varSviPredmetiItemId known) |
| Dokumenti file | Does not exist |
| Registry counter | Incremented |
| Reserved number | Validated but not deleted |

**Failure handling (e.g. main file Create file fails):**

The Svi predmeti item exists but has no associated file.
This is a partially created document — the registry number is assigned but the document content is missing.

**Compensation action:**
1. Attempt to **delete the Svi predmeti item** (compensation delete):
   ```
   Action: SharePoint — Delete item
   List: EV_DocCentralV3_lstSviPredmeti
   ID: varSviPredmetiItemId
   ```
2. If compensation delete succeeds: log the rollback. Return CREATE_FILE_FAILED.
   State is now equivalent to the previous clean-after-number-acquired state.
3. If compensation delete fails:
   - Log Critical: `Svi predmeti item (ID: varSviPredmetiItemId) nije obrisano pri kompenzaciji. DelovodniBroj: varDelovodniBroj. Potrebna ručna intervencija.`
   - Return CREATE_FILE_FAILED with the itemId in the response (to help the administrator find it).

**Important:** Compensation delete must be attempted inside the Catch scope, not in the Try scope.
Add the delete action at the start of the Catch scope body, before the LogEvent call.

For reserved number path: the reserved number item was not deleted (deletion is Step 5l, which was never reached). It remains available.

---

### After Step 5h — main file created (before metadata update)

| Artifact | State |
|---|---|
| Svi predmeti item | Exists |
| Main file | **Exists** (varMainFileId known) |
| File metadata | Incomplete (system name, link to Svi predmeti not written) |

**Failure handling (metadata update fails):**
This is **non-fatal**. The file exists in Dokumenti. The Svi predmeti item exists.
The link between them via metadata is missing, but both artifacts are correct.
An administrator can manually set the metadata columns.

Log Warning. Continue. Do not roll back.

---

### After Step 5j — attachments processed (some may have failed)

Attachment failures are non-fatal throughout. The document is considered fully created
even if one or more attachments are missing. Log Warning per failed attachment.

---

### After Step 5k — permissions assigned (success or non-fatal failure)

Permission failure does not trigger compensation. The document is created.
Log Warning if permissions failed. Return success with warning message.

---

### After Step 5l — reserved number deleted (or failed to delete)

Reserved number deletion failure is non-fatal. The document is fully created.
An orphaned reserved number entry in RezervisaniBrojevi must be cleaned up manually.

The administrator should:
1. Find the AuditLog Warning entry for the failed deletion.
2. Verify the reserved number's delovodniBroj matches a real document in Svi predmeti.
3. Manually delete the RezervisaniBrojevi item.

---

## Catch scope compensation design

The Catch scope in this flow must:

1. **Check whether Svi predmeti item was created** (varSviPredmetiItemId > 0).
   - If yes: attempt compensation delete.
   - If no: no compensation needed — log and return.

2. **Attempt compensation delete** (if varSviPredmetiItemId > 0):
   - If the main file also exists (varMainFileId > 0): optionally attempt to delete the main file too.
     File deletion in Dokumenti via SharePoint Delete item on library list.
   - Compensation delete failures must be logged as Critical — not silently swallowed.

3. **Log the error** with full context.

4. **Respond failure** to Canvas App.

Power Automate Catch scope example structure:
```
[Catch Scope]
├── Condition: varSviPredmetiItemId > 0
│   └── Yes:
│       ├── Delete item (Svi predmeti, varSviPredmetiItemId)
│       ├── Condition: Delete succeeded?
│       │   ├── Yes: Log "Kompenzaciono brisanje uspešno."
│       │   └── No: Log Critical "Kompenzaciono brisanje nije uspelo."
│       └── [Optional] Delete file (Dokumenti, varMainFileId) if varMainFileId > 0
├── Log CreateDocument / Error / Failed / CREATE_DOCUMENT_FAILED
└── Respond failure
```

---

## Registry number orphan policy

When a generated registry number is acquired (counter incremented in App Config) but the
document is never successfully created, the number is "lost" — it will not be re-used.
The next generation call will get counter+1, skipping the lost number.

This is **intentional and acceptable**:
- Registry number sequences do not need to be gapless. Gaps are normal in any registry system.
- Attempting to "return" a number by decrementing the counter creates race conditions.
- The AuditLog entry records the lost number for reconciliation if needed.

Administrators must not manually adjust the App Config counter to fill gaps.

---

## Summary: what requires manual administrator action

| Scenario | Required action |
|---|---|
| Registry number generated, Svi predmeti item not created | Verify in AuditLog. No document exists. No action needed (gap in sequence is acceptable). |
| Svi predmeti item created, compensation delete succeeded | No document exists. Retry is safe. |
| Svi predmeti item created, compensation delete failed | Critical log. Administrator must manually delete the orphan Svi predmeti item and associated file (if created). |
| Main file created, metadata not written | Administrator manually updates file metadata columns in SharePoint. |
| Attachment not created or metadata missing | Administrator can re-upload the attachment manually and set metadata. |
| Permissions incomplete | Administrator calls AssignPermissions flow manually or sets permissions via SharePoint UI. |
| Reserved number orphaned (deletion failed) | Administrator manually deletes the RezervisaniBrojevi item after verifying the document exists. |
