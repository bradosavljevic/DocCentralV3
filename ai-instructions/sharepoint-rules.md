# SharePoint Rules

## Core lists/libraries

- Svi predmeti
- Dokumenta
- EmailDocuments
- AppConfig
- Rezervisani brojevi
- Partneri
- Podsetnici
- AuditLog
- NumberLocks / ZakljucavanjeBrojeva

## User permissions

Users are Read-only. Writes are done by Power Automate service account.

## Indexes and unique constraints

Required:
- `Svi predmeti.DelovodniBroj`: indexed + unique
- `Rezervisani brojevi.RezervisaniBroj`: indexed + unique
- `NumberLocks.LockKey`: indexed + unique

Recommended:
- Partneri.PIB indexed if used for lookup/import
- Stanje indexed
- DatumZavodjenja indexed
- TipDokumenta indexed
- OrganizacionaJedinica indexed

## Internal names

Claude must not guess internal names. Human must return internal names before formulas/flows use them.
