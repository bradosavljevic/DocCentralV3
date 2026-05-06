# 07 - Power Automate Rules

## Manual implementation

Claude Code does not create Power Automate flows directly. For every flow, Claude must create a manual implementation guide in `flow-guides/`.

## Service account

All write flows must use service account connections.

## Standard flow structure

Critical flows should use:
- Try scope
- Catch scope
- Finally scope
- AuditLog write
- clear response to Power Apps where relevant

## Error handling

- user-friendly message to Canvas app
- technical detail to AuditLog
- no silent failures
- do not show raw technical errors to business users unless needed for support

## AuditLog

Critical flows should log successful registry number assignment, failed registry number assignment, reserved number use, reserved number delete failure, AppConfig update, archive book generation, year closing, flow errors and recovery actions.

## OneDrive

OneDrive actions may be used when needed. If temporary files are created, include cleanup action and recovery scenario for cleanup failure.

## Large list read flows

Use read flows for large lists with OData filters, selected columns, pagination and JSON response.
