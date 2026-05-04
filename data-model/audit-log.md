# SharePoint lista: Audit / Log

## Svrha

Centralna SharePoint lista za logovanje bitnih događaja, grešaka i tehničkih aktivnosti.

## Obavezno logovati

- neuspešno kreiranje dokumenta
- pokušaj generisanja delovodnog broja
- uspešno generisanje delovodnog broja
- neuspešno generisanje broja
- korišćenje rezervisanog broja
- arhiviranje
- zaključenje godine
- grešku Power Automate flow-a

## Preporučena polja

- ID
- Title
- EventType
- EventStatus
- Severity
- CorrelationId
- FlowName
- FlowRunId
- UserEmail
- UserDisplayName
- DocumentId
- DelovodniBroj
- Godina
- PayloadJson
- ErrorMessage
- ErrorDetails
- Created

## Severity vrednosti

- Info
- Warning
- Error
- Critical

## EventStatus vrednosti

- Started
- Success
- Failed
- Retried
- Skipped

## Napomena

Audit se implementira kroz SharePoint liste, ne kroz Dataverse ili eksterni logging sistem.
