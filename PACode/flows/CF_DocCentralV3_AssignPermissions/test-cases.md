# Test cases: CF_DocCentralV3_AssignPermissions

## Prerequisites before testing

- A Svi predmeti item exists for testing (note its item ID).
- A file exists in the Dokumenti library (note its item ID).
- At least one test user account is available whose email can be resolved via ensureuser.
- At least one Entra group ID is known and has been tested for resolution (Pattern A or B from `sharepoint-rest-permissions.md`).
- App Config has a default org unit group entry (key UNKNOWN — resolve before testing).
- CF_DocCentralV3_LogEvent is working.
- All UNKNOWN items in `sharepoint-rest-permissions.md` and `permission-model.md` have been resolved.

## How to test

Create a temporary instant cloud flow with "Run a Child Flow" targeting CF_DocCentralV3_AssignPermissions.

After each test, verify permissions using:
1. SharePoint → Svi predmeti → open item → Info pane → Manage access.
2. SharePoint → Dokumenti → open file → Info pane → Manage access.
3. AuditLog list — check for PermissionAssignment entries.
4. Optionally: use the diagnostic REST call from `sharepoint-rest-permissions.md` (section 7) to read role assignments via API.

---

## TC-001: Happy path — initiator + org unit group + one file

**Purpose:** Verify that inheritance is broken and initiator and org unit group both get Read on the item and file.

**Input:**
```json
{
  "sviPredmetiItemId": <testItemId>,
  "documentLibraryFileIds": "[<testFileId>]",
  "initiatorEmail": "test.user@organizacija.rs",
  "documentType": "Ulazni",
  "orgUnitGroupId": "<testGroupObjectId>",
  "additionalUserEmails": "[]",
  "additionalGroupIds": "[]",
  "correlationId": "test-correlation-001"
}
```

**Expected response:**
```json
{
  "success": true,
  "message": "Prava su uspešno dodeljena.",
  "correlationId": "test-correlation-001",
  "errorCode": ""
}
```

**Post-test verification:**
- Svi predmeti item: inheritance is broken. Permissions show: service account (Full Control), test.user (Read), org unit group (Read).
- Dokumenti file: same permission set.
- AuditLog: 1 × PermissionAssignment / Info / Started, 1 × PermissionAssignment / Info / Success.

**Pass criteria:** Flow green. Permissions match expected. No inherited permissions shown.

---

## TC-002: No org unit group — fallback to App Config default

**Purpose:** Verify that when orgUnitGroupId is empty, the flow reads the default from App Config.

**Input:** Same as TC-001 but with `"orgUnitGroupId": ""`.

**Expected:**
- Flow reads default group from App Config.
- If default group is found: that group gets Read.
- AuditLog: no Warning about missing group.

**Pass criteria:** Default org unit group has Read on item and file. Flow returns success.

---

## TC-003: No org unit group — App Config default also missing

**Purpose:** Verify that when orgUnitGroupId is empty AND App Config has no default, the flow logs a warning but continues.

**Setup:** Temporarily remove or blank the default org unit group entry in App Config.

**Expected:**
- Flow returns success (non-fatal).
- Svi predmeti item permissions: initiator Read only (plus service account Full Control).
- AuditLog: 1 × Warning log entry about missing org unit group.

**Restore:** Reinstate the App Config default after test.

**Pass criteria:** Success returned even without any group. Warning in AuditLog.

---

## TC-004: Additional approver user passed

**Purpose:** Verify that an approver email in additionalUserEmails gets Read on the item.

**Input:**
```json
{
  "sviPredmetiItemId": <testItemId>,
  "documentLibraryFileIds": "[]",
  "initiatorEmail": "test.user@organizacija.rs",
  "documentType": "Ulazni",
  "orgUnitGroupId": "<testGroupObjectId>",
  "additionalUserEmails": "[\"odobravalac@organizacija.rs\"]",
  "additionalGroupIds": "[]",
  "correlationId": "test-correlation-004"
}
```

**Expected:**
- Svi predmeti item has Read for: initiator, org unit group, odobravalac.
- No file permissions (documentLibraryFileIds is empty).

**Pass criteria:** Three principals have Read. No file permissions changed.

---

## TC-005: Additional approver group passed

**Purpose:** Verify that a group in additionalGroupIds gets Read.

**Input:** Same as TC-004 but with:
```json
"additionalUserEmails": "[]",
"additionalGroupIds": "[\"<testGroupObjectId2>\"]"
```

**Expected:** org unit group and additional group both have Read.

**Pass criteria:** Two groups have Read on item. Flow green.

---

## TC-006: Multiple files in documentLibraryFileIds

**Purpose:** Verify that all files in the array receive the same permissions.

**Setup:** Two test files exist in Dokumenti. Note their item IDs.

**Input:** `"documentLibraryFileIds": "[<fileId1>, <fileId2>]"`

**Expected:** Both files have inheritance broken and same Read grants.

**Pass criteria:** Both file permission sets match Svi predmeti item. Flow green.

---

## TC-007: Invalid user email — ensureuser fails

**Purpose:** Verify that a non-existent user in additionalUserEmails causes a Warning (non-fatal) and the flow continues.

**Input:**
```json
"additionalUserEmails": "[\"nonexistent.user.XXXX@organizacija.rs\"]"
```

**Expected:**
- Flow returns success (non-fatal).
- AuditLog: Warning / PERMISSION_USER_RESOLVE_FAILED for the failed email.
- Other principals (initiator, org unit group) still receive Read.

**Pass criteria:** Success returned. Warning in AuditLog. Other permissions applied correctly.

---

## TC-008: Empty additionalUserEmails and additionalGroupIds

**Purpose:** Verify that passing empty arrays `"[]"` does not cause errors.

**Input:** Both additionalUserEmails and additionalGroupIds = `"[]"`.

**Expected:** Only initiator and org unit group get Read. No loop iteration errors.

**Pass criteria:** Flow green. No error from empty array processing.

---

## TC-009: Break inheritance fails (simulated)

**Purpose:** Verify that PERMISSION_BREAK_FAILED triggers the Catch scope and returns a structured failure.

**Simulation:** Remove Full Control from the service account on the test Svi predmeti item
so it cannot manage permissions, then run the flow.

**Expected:**
- Flow returns failure: `{ "success": false, "errorCode": "PERMISSION_BREAK_FAILED" }`
- AuditLog: PermissionAssignment / Error / Failed / PERMISSION_BREAK_FAILED.

**Restore:** Re-grant Full Control to service account after test.

**Pass criteria:** Structured failure returned. No unhandled exception.

---

## TC-010: Empty correlationId — auto-generation

**Purpose:** Verify GUID auto-generation when correlationId is empty.

**Input:** `"correlationId": ""`

**Expected:** Response correlationId is a valid GUID. AuditLog entries share that GUID.

**Pass criteria:** Valid GUID in response. AuditLog entries linked by same correlationId.

---

## TC-011: Re-application overwrites previous permissions

**Purpose:** Verify that calling this flow twice on the same item replaces (not accumulates) permissions.

**Method:**
1. Run TC-004 (grants initiator + org unit group + odobravalac Read).
2. Run TC-001 on the same item (no additionalUserEmails).
3. Check item permissions after step 2.

**Expected:** After step 2, odobravalac no longer has Read. Only initiator + org unit group.

**Pass criteria:** Permissions match the second call's inputs exactly. odobravalac's Read is gone.

---

## TC-012: Verify parent flow non-fatal handling

**Purpose:** Verify that CF_DocCentralV3_CreateDocument continues even if this flow returns failure.

**Method:**
1. Simulate a permission failure (TC-009 setup).
2. Call this flow from a test version of CF_DocCentralV3_CreateDocument.
3. Verify that CF_DocCentralV3_CreateDocument returns success to the Canvas App (with a warning message about permissions).

**Pass criteria:** Document item created. Parent flow returns success. AuditLog shows permission failure. Canvas App receives a warning message.

---

## Post-test cleanup

1. Restore service account permissions if changed for TC-009.
2. Restore App Config default org unit group if changed for TC-003.
3. Optionally reset Svi predmeti item and Dokumenti file permissions to library defaults (restore inheritance).
4. Delete test Svi predmeti items or files if created solely for testing.
5. Confirm the flow is inside the DocCentralV3 solution.
