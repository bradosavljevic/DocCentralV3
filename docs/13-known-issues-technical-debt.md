# 13 - Known Issues And Technical Debt

## Direct SharePoint writes from Canvas

Existing app contains direct Patch/Remove patterns that must be removed or isolated.

## Registry number race condition

Existing solution uses a separate flow for number assignment to reduce race condition. Target design must include NumberLocks, no-skipped-number rule, recovery blocking, AuditLog and unique/indexed DelovodniBroj.

## AppConfig duplicate e-invoice config

There should be only `EfaktureParametri`. Legacy/duplicate `EFaktureParametri` must not be deleted automatically without migration.

## Large scrCodes screen

`scrCodes` is too large and should be refactored.

## AppChecker

Known areas: accessibility labels, TabIndex, focus border and screen/control complexity.

## Large SharePoint lists

Partneri, Svi predmeti, Dokumenta and archiving datasets may exceed 2,000 items. Use flow-based read patterns.
