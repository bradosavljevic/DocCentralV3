# Power Automate Checklist

## Osnovno

- [ ] Naziv flow-a
- [ ] Trigger
- [ ] Konektori
- [ ] Vlasnik
- [ ] Connection reference
- [ ] Environment variables

## Trigger

- [ ] Trigger condition
- [ ] Concurrency control
- [ ] Retry policy
- [ ] Timeout
- [ ] Trigger input schema

## Akcije

- [ ] SharePoint akcije
- [ ] Approval akcije
- [ ] HTTP akcije
- [ ] Compose / Select / Filter array
- [ ] Parse JSON
- [ ] Apply to each
- [ ] Conditions
- [ ] Scopes

## Error handling

- [ ] Try scope
- [ ] Catch scope
- [ ] Finally scope
- [ ] Configure run after
- [ ] Error log
- [ ] Response ka Power Apps u grešci
- [ ] Audit log za uspešne operacije

## Response

- [ ] Jasni output parametri
- [ ] Status
- [ ] Message
- [ ] ErrorCode
- [ ] ItemId
- [ ] DocumentNumber
- [ ] CorrelationId / RunId

## Rizici

- [ ] Flow traje predugo za Power Apps response
- [ ] Nema child flow podela
- [ ] Nema centralizovanog logovanja
- [ ] Nema concurrency zaštite
- [ ] Nema retry logike za kritične upise
