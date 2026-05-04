# Skill: Power Automate

## Standardi

- Service account connections
- Controlled input/output
- Error handling scopes
- Audit logging
- CorrelationId
- Retry logic za kritične operacije
- User-friendly error response
- Technical error details u log listi

## Kritični flow-ovi

- RegisterDocument
- GenerateOrReserveRegistryNumber
- StartApprovalProcess
- ProcessApprovalResponse
- SendDailyReminders
- ArchiveDocuments
- CloseRegistryYear
- ExportConfigurationToExcel
- GenerateArchiveBookPdf

## Pravila

- Create/Edit/Delete radi flow.
- Reserved number se ne briše pre uspešnog kreiranja dokumenta.
- Brojač se ne uvećava ako zavođenje nije uspešno.
- Flow mora vratiti kontrolisan response aplikaciji.


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
