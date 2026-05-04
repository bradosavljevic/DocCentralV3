# 08 — Numbering and Concurrency

## Ključni zahtev

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

## Potvrđeno

- App Config sadrži `Delovodne knjige`.
- U App Config se čuva sledeći delovodni broj i povećava se.
- Postoji lista `RezervisaniBrojevi`.
- Postoji ekran `scrReserveNumber`.
- Postoje flow-ovi `CF_DocCentral21_ReserveNumber` i `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`.
- Svake godine se zaključava delovodnik i kreira arhivska knjiga za zavedene dokumente u statusu arhivirano.
- ETag se trenutno ne koristi.
- Retry logika i audit neuspelih pokušaja nisu potvrđeni i treba ih projektovati.

## Target dizajn

- Backend Numbering Service.
- ETag / optimistic locking.
- Retry-on-conflict.
- Unique constraint.
- Audit log.
- Error queue.
- Correlation ID.

## Zaključak

Canvas App ne sme biti izvor istine za delovodni broj.
