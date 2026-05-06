# AppConfig Rules

AppConfig stores JSON dictionaries and configuration.

## Update rule

All AppConfig changes must go through Power Automate.

## Schema validation

Every update must validate JSON schema. Invalid schema must be rejected.

## GUID IDs

Every row inside AppConfig JSON arrays should have `ID` as GUID where possible.

## Excel import

Import is per dictionary:
- insert missing records
- update changed records
- leave unchanged records
- log changes

## Logging

AppConfig updates may log ConfigKey, old JSON, new JSON, old hash, new hash, changed by, changed at and validation result.
