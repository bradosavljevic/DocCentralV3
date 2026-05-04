# Skill: Power Automate Logging and Audit

## Cilj

Enterprise verzija mora imati centralno logovanje i audit kritičnih operacija.

## Šta logovati

- kreiranje dokumenta;
- izmenu dokumenta;
- zavođenje dokumenta;
- rezervaciju delovodnog broja;
- neuspešnu rezervaciju broja;
- arhiviranje;
- promenu partnera;
- promenu prava pristupa;
- greške u flow-ovima;
- approval odluke;
- neuspešne pokušaje importovanja ako se import kasnije uključi.

## Minimalna polja loga

- `Timestamp`
- `CorrelationId`
- `FlowName`
- `Operation`
- `EntityName`
- `EntityId`
- `UserEmail`
- `Status`
- `ErrorCode`
- `ErrorMessage`
- `InputSummary`
- `OutputSummary`
- `DurationMs`

## Pravila

- Ne logovati osetljive podatke nepotrebno.
- Ne logovati kompletne dokumente u plain text.
- Log mora pomoći u dijagnostici, ali ne sme ugroziti bezbednost.
- Svaki flow iz Canvas aplikacije mora vratiti `correlationId`.

## Preporuka

Ako Dataverse nije dostupan ili se izbegava premium, koristiti SharePoint listu za audit log, uz pažljivo indeksiranje i retention politiku.
