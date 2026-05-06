# Power Automate Coding Standards

Claude writes implementation guides. Human builds flows manually.

## Connections

Write flows must use service account connections.

## Flow structure

Recommended:
- Initialize input variables
- Try scope
- Catch scope
- Finally scope
- Respond to Power Apps if applicable

## Error handling

Catch scope should compose technical error, write AuditLog, return user-friendly message and preserve sequence safety for registry numbers.

## Number assignment

Never implement number assignment without NumberLocks, no-skip rule, recovery blocking, unique/indexed DelovodniBroj and AuditLog.
