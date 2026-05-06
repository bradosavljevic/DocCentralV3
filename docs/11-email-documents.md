# 11 - Email Documents

`EmailDocuments` is a staging library for files received by email.

## Intake process

1. Email arrives in shared mailbox.
2. Flow saves attachments/files to `EmailDocuments`.
3. No `Svi predmeti` item is created yet.
4. User responsible for registration opens unprocessed documents screen.
5. User enters required metadata.
6. File is copied to `Dokumenta`.
7. `Svi predmeti` item is created.
8. Registry number process continues.

## Cleanup

Processed email documents do not need immediate deletion. Use scheduled cleanup. Retention period should be configurable in AppConfig. Recommended default is 30 days after processing.

## Future scope

Teams notifications may be added later, but current scope uses email only.
