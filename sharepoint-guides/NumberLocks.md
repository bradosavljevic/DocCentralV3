# SharePoint Guide: NumberLocks / ZakljucavanjeBrojeva

## Purpose

Prevents parallel registry number assignment.

## Recommended list name

`NumberLocks` or `ZakljucavanjeBrojeva`

## Columns

| Display name | Internal name suggestion | Type | Required | Notes |
|---|---|---|---|---|
| Title | Title | Single line text | Yes | Same as LockKey |
| LockKey | LockKey | Single line text | Yes | Unique + indexed |
| LockedBy | LockedBy | Single line text | No | User/service/flow |
| LockedAt | LockedAt | DateTime | Yes | UTC |
| ExpiresAt | ExpiresAt | DateTime | Yes | UTC + 5 minutes |
| FlowRunId | FlowRunId | Single line text | No | Current run |
| Status | Status | Choice | Yes | Active, Released, Expired, Failed |
| RelatedItemId | RelatedItemId | Number | No | Svi predmeti ID |
| RelatedDocumentId | RelatedDocumentId | Number | No | Dokumenta item ID |
| ErrorCode | ErrorCode | Single line text | No | For recovery |

## Required settings

- `LockKey` must be unique.
- `LockKey` must be indexed.
- Lock duration: 5 minutes.
