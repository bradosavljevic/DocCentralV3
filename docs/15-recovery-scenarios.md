# 15 - Recovery Scenarios

## Recovery principle

Partially failed processes must be detected, logged and recoverable. If registry number continuity is at risk, all new registrations must be blocked until recovery is complete.

## REC-001: Document created without DelovodniBroj

State:
- file exists in Dokumenta
- item exists in Svi predmeti
- DelovodniBroj missing

Expected:
- TechnicalStatus = FailedNeedsRecovery
- AuditLog entry
- new registrations blocked
- admin/consultant recovery button available
- no next number assignment until resolved

## REC-002: DelovodniBroj in Svi predmeti but not in Dokumenta

State:
- Svi predmeti has DelovodniBroj
- Dokumenta metadata missing DelovodniBroj

Expected:
- Svi predmeti remains source of truth
- AuditLog entry
- recovery flow updates Dokumenta metadata
- do not assign new number until sequence safety is confirmed

## REC-003: Failed deletion of used reserved number

Expected:
- AuditLog entry
- recovery attempts delete again
- prevent duplicate use of reserved number

## REC-004: Archive book PDF created but not registered

Expected:
- AuditLog
- recovery flow completes registration
- duplicate generation blocked

## REC-005: AppConfig update failed

Expected:
- AuditLog
- stop if number sequence risk exists
- recovery updates AppConfig or resolves manually

## REC-006: Temporary file cleanup failed

Expected:
- no business data loss
- AuditLog warning
- scheduled cleanup removes file later
