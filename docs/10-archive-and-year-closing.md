# 10 - Archive And Year Closing

## Archiving

Direct transition `Zavedeno -> Arhivirano` is allowed if the document is not in process. If document goes through approval, after approval process finishes the document returns to `Zavedeno`.

## Archive book

Archive book:
- generated once per year
- contains all documents for that year
- cannot be regenerated if already exists
- cannot have a new version
- has its own `DelovodniBroj`
- uses same registry book/counter as other documents
- has its own document type and category
- is stored as physical PDF
- is registered in `Svi predmeti`
- has file/metadata in `Dokumenta`

## Year closing prerequisites

Before year closing:
- all documents must be `Arhivirano`
- no documents may be `U odobravanju`
- no documents may be `Odbijeno`
- reserved numbers list must be empty because all reserved numbers must be used
- archive book must be generated and registered
- active year must be updated in AppConfig

## Closing year

- manual action
- closed year can never be reopened
- do not create new `DelovodneKnjige` item
- update existing AppConfig JSON
- set new active year
- reset counter to 1
