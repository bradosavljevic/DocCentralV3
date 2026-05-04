# Numbering Sequence

## Cilj

Definisati target sekvencu za concurrency-safe dodelu delovodnog broja.

| Event | Kada se upisuje |
|---|---|
| `RegistrationStarted` | Kada Canvas pokrene proces |
| `PayloadValidated` | Nakon validacije |
| `NumberReservationStarted` | Pre pokušaja rezervacije |
| `NumberReservationRetry` | Svaki retry |
| `NumberReserved` | Kada je broj rezervisan |
| `DocumentUploaded` | Kada je dokument sačuvan |
| `MetadataUpdated` | Kada je metadata upisan |
| `NumberCommitted` | Kada je broj potvrđen |
| `RegistrationCompleted` | Kada je proces završen |
| `RegistrationFailed` | Kada proces padne |

## Povezani postojeći elementi

```text
scrReserveNumber
CF_DocCentral21_ReserveNumber
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
CF_DocCentral21_ResetDelovodnika
CF_DocCentral21_LockYear
CF_DocCentral21_GetLockYearData
RezervisaniBrojevi
App Config / Delovodne knjige
```
