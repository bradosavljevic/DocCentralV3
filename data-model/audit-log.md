# SharePoint lista: AuditLog

## Environment variable

```text
EV_DocCentralV3_lstAuditLog
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstAuditLog
```

## Namena

Lista `AuditLog` služi za logovanje bitnih aktivnosti i grešaka u sistemu.

Audit log se koristi za:

- dijagnostiku
- praćenje neuspešnih operacija
- praćenje generisanja delovodnog broja
- praćenje korišćenja rezervisanih brojeva
- praćenje arhiviranja
- praćenje zaključenja godine
- praćenje Power Automate grešaka

## Obavezni događaji za logovanje

- neuspešno kreiranje dokumenta
- pokušaj generisanja delovodnog broja
- uspešno generisan delovodni broj
- neuspešno generisanje broja
- korišćenje rezervisanog broja
- arhiviranje
- zaključenje godine
- greška Power Automate flow-a

## Preporučena struktura kolona

| Display name | Tip | Obavezno | Opis |
|---|---|---:|---|
| Title | Single line of text | Da | Kratak naziv događaja |
| EventType | Choice | Da | Tip događaja |
| EventCategory | Choice | Da | Kategorija događaja |
| Severity | Choice | Da | Info, Warning, Error, Critical |
| Status | Choice | Da | Success, Failed, Started, Retried |
| CorrelationId | Single line of text | Da | ID procesa/flow run-a |
| FlowName | Single line of text | Ne | Naziv flow-a |
| FlowRunId | Single line of text | Ne | ID izvršavanja flow-a |
| Source | Choice | Da | CanvasApp, CloudFlow, SharePoint, System |
| UserEmail | Single line of text | Ne | Korisnik koji je izazvao događaj |
| UserDisplayName | Single line of text | Ne | Ime korisnika |
| DocumentItemId | Number | Ne | ID itema u Svi predmeti |
| DelovodniBroj | Single line of text | Ne | Delovodni broj |
| ReservedNumberId | Number | Ne | ID rezervisanog broja |
| PartnerId | Number | Ne | ID partnera |
| Message | Multiple lines of text | Da | Kratka poruka |
| DetailsJson | Multiple lines of text | Ne | Tehnički detalji u JSON formatu |
| ErrorCode | Single line of text | Ne | Kod greške |
| ErrorMessage | Multiple lines of text | Ne | Poruka greške |
| RequestPayloadJson | Multiple lines of text | Ne | Ulazni payload |
| ResponsePayloadJson | Multiple lines of text | Ne | Izlazni payload |
| CreatedAt | Date and time | Da | Datum i vreme događaja |
| EnvironmentName | Single line of text | Ne | Naziv environment-a |
| SolutionName | Single line of text | Ne | Naziv solution-a |

## Choice vrednosti

### EventType

```text
CreateDocument
GenerateRegistryNumber
UseReservedNumber
ArchiveDocument
CloseRegistryYear
SendForApproval
ProcessApprovalResponse
SendReminder
GenerateArchiveBookPdf
PermissionAssignment
PowerAutomateError
ValidationError
SystemError
```

### EventCategory

```text
Document
RegistryNumber
ReservedNumber
Approval
Reminder
Archive
YearClosing
Permission
Partner
System
```

### Severity

```text
Info
Warning
Error
Critical
```

### Status

```text
Started
Success
Failed
Retried
Skipped
```

### Source

```text
CanvasApp
CloudFlow
SharePoint
System
```

## Napomena

Audit log treba pisati iz flow-a:

```text
CF_DocCentralV3_LogEvent
```

Ostali flow-ovi treba da pozivaju ovaj flow ili da koriste istu standardizovanu strukturu za logovanje.
