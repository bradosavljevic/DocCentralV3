# Flow Guide: CF_DocCentral_CleanupAuditLog

## Purpose

Scheduled weekly cleanup of AuditLog entries older than 6 months.

## Trigger

Power Apps V2 unless otherwise specified.

## Connections

Use service account connection references.

## Inputs

To be defined in implementation.

## Outputs

Recommended response:

```json
{
  "success": true,
  "code": "OK",
  "message": "Operation completed.",
  "data": {}
}
```

## Manual implementation steps

1. Create flow with the exact name.
2. Configure trigger.
3. Add input parameters.
4. Add Try scope.
5. Add required SharePoint actions.
6. Add validation.
7. Add AuditLog write where critical.
8. Add Catch scope.
9. Add Finally scope.
10. Add response to Power Apps if applicable.
11. Test.
12. Return final schema to Claude.

## Error handling

- user-friendly message
- technical details to AuditLog
- no silent failures

## Test cases

- success
- validation failure
- SharePoint connection failure
- permission failure
- partial failure
