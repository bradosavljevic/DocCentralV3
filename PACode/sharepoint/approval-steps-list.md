# DEPRECATED: ApprovalSteps list

## Status: DEPRECATED

This document is superseded by the updated approval mechanism decision.

**Do not create this list.**
**Do not create `EV_DocCentralV3_lstApprovalSteps`.**

## Reason for deprecation

The approval process tracking was redesigned to use `Svi predmeti` fields directly,
avoiding the need for a separate SharePoint list.

Approval workflow configuration is driven by **ProcesConfig** in `App Config`,
loaded into `colProcesConfig` by the Canvas App at startup.

Approval runtime state (current approver, step progress, history) is stored in
proposed JSON fields on `Svi predmeti` items.

## Reference

See updated design in:

- `PACode/decisions/approval-mechanism.md`
- `PACode/flows/CF_DocCentralV3_SendForApproval-design.md`
- `PACode/flows/CF_DocCentralV3_ProcessApprovalResponse-design.md`
