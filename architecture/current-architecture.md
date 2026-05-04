# Current Architecture

## Status

```text
DOPUNJENO NA OSNOVU POWER PLATFORM SOLUTION-A
```

## Sažetak

Trenutna arhitektura:

```text
Power Apps Canvas App + Power Automate + SharePoint Online
```

## Solution nalazi

| Oblast | Nalaz |
|---|---|
| Canvas aplikacija | 1 |
| Ekrani | 16 |
| Flow-ovi | 20 |
| Environment variables | 9 |
| Connection references | 5 |

## Komponente

- UI: Canvas aplikacija.
- Backend: Power Automate.
- Data/storage: SharePoint Online.
- Config: AppConfig.
- Numbering: App Config + RezervisaniBrojevi + flow-ovi.
- Email intake: EmailDocuments + Outlook/shared mailbox flow-ovi.
- Export: Exports + `CF_DocCentral21_ExportCodes`.

## Najveći rizik

```text
Dodela delovodnog broja mora biti concurrency-safe.
```
