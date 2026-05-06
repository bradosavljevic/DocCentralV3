# SharePoint Guide: AuditLog

## Purpose

Stores important business and technical events.

## Retention

6 months. Scheduled cleanup flow runs weekly.

## Suggested columns

| Display name | Type | Notes |
|---|---|---|
| Title | Single line text | Short event title |
| EventType | Single line text / Choice | e.g. NumberAssigned |
| Severity | Choice | Info, Warning, Error, Critical |
| RelatedItemId | Number | Svi predmeti item |
| RelatedDocumentId | Number | Dokumenta item |
| FlowName | Single line text | Flow name |
| FlowRunId | Single line text | Run id |
| UserEmail | Single line text | Initiating user |
| UserMessage | Multiple lines | Message for user |
| TechnicalDetails | Multiple lines | Technical error/details |
| OldJson | Multiple lines | optional |
| NewJson | Multiple lines | optional |
