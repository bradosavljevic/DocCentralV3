# 14 - Test Matrix

## Document registration

| ID | Scenario | Expected result |
|---|---|---|
| REG-001 | Register document with valid main file | Document and item created, number assigned |
| REG-002 | Register without main file | Registration blocked |
| REG-003 | Register with reserved number | Reserved number used and deleted after success |
| REG-004 | Register with optional attachments | Attachments stored with same DelovodniBroj and Attachment=true |
| REG-005 | 10 users register at same time | 10 sequential unique numbers, no skips |
| REG-006 | Number assignment fails | New registrations blocked until recovery |

## Reserved numbers

| ID | Scenario | Expected result |
|---|---|---|
| RN-001 | Create reserved number | Number appears in list |
| RN-002 | Use reserved number | Number deleted only after Svi predmeti and Dokumenta update |
| RN-003 | Year closing with reserved numbers | Closing blocked |
| RN-004 | Reserved number from active year | Allowed |
| RN-005 | Reserved number for any document type | Allowed |

## Approval

| ID | Scenario | Expected result |
|---|---|---|
| APR-001 | TipDokumenta has active ProcesConfig | Document enters configured process |
| APR-002 | No active process | Document becomes Zavedeno |
| APR-003 | User approval | Step completed by user |
| APR-004 | Group approval | First group response completes step |
| APR-005 | Rejection | Document becomes Odbijeno and returns to initiator |
| APR-006 | Restart after rejection | Initiator can edit and restart |

## Email documents

| ID | Scenario | Expected result |
|---|---|---|
| EML-001 | Email attachment arrives | File saved to EmailDocuments only |
| EML-002 | Process email document | File copied to Dokumenta and item created |
| EML-003 | Cleanup processed files | Old processed files deleted by scheduled flow |

## Archive and year closing

| ID | Scenario | Expected result |
|---|---|---|
| ARC-001 | Archive Zavedeno document | Status becomes Arhivirano |
| ARC-002 | Archive document in process | Blocked |
| ARC-003 | Generate archive book | PDF generated and registered |
| ARC-004 | Generate archive book twice | Blocked |
| YEAR-001 | Close year with non-archived docs | Blocked |
| YEAR-002 | Close year with reserved numbers | Blocked |
| YEAR-003 | Close valid year | Active year updated, counter reset |
| YEAR-004 | Reopen closed year | Not allowed |
