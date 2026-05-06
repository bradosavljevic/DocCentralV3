# 09 - Approval Process

## Source of truth

`ProcesConfig` is the source of truth. Process is resolved by `TipDokumenta`.

## No active process

If no active process exists, document is registered as `Zavedeno` and later it can be manually archived after archive metadata is completed.

## Approval types

Approval step can target one user or one group.

For group approval:
- all users may receive notification
- first valid response completes the step
- process continues to next step

## Approval modes

Configurable approval modes:
- `App`
- `PowerAutomateApprovals`

## Rejection

If rejected:
- status becomes `Odbijeno`
- document returns to initiator
- initiator can edit metadata
- initiator can restart process

## History

Approval history may be stored in `ApprovalHistoryJson`, AuditLog or another dedicated structure if needed. Retention should not exceed 6 months where it is used only as process/history log.
