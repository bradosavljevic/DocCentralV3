# Power Apps — Navigation Model

## Status

```text
DOPUNJENO NA OSNOVU IDENTIFIKOVANIH EKRANA
```

## Pretpostavljeni glavni navigacioni model

```text
scrHome
├── scrAddDocument
├── scrReserveNumber
├── scrArchive
├── scrCodes
├── scrArchiveBook
│   └── scrArchiveBookItems
├── scrLockYear
├── scrSettings
├── scrPartneri
├── scrEDocuments
├── scrEmailDocs
│   └── scrEmail
├── scrDocSwap
├── scrAddReminders
└── scrProcess
```

## Nepoznato

- Tačne `Navigate(...)` formule.
- Deep link parametri.
- Role-based navigation.
- Koji ekrani rade refresh kolekcija na `OnVisible`.

## Preporuke

- Napraviti centralizovani navigation config.
- Vidljivost menija vezati za backend/Entra autorizaciju, ne samo UI.
