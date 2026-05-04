# Skill: Power Automate Error Handling

## Cilj

Svaki flow mora imati predvidiv error handling i retry strategiju.

## Standardna struktura

Koristiti Scope pattern:

- `Scope - Try`
- `Scope - Catch`
- `Scope - Finally`

## Catch mora da uradi

- uhvati grešku;
- upiše log;
- vrati strukturisan error response;
- očisti privremene rezervacije ako je potrebno;
- ne ostavi dokument u nekonzistentnom stanju.

## Retry

Za prolazne greške koristiti kontrolisan retry:

- SharePoint throttling;
- conflict kod unique constraint-a;
- konflikt kod delovodnog broja;
- privremena greška konektora.

Retry ne sme beskonačno da traje.

## Delovodni broj

Kod dodele delovodnog broja:

- ako rezervacija ne uspe zbog konflikta, pokušati ponovo sa sledećim brojem;
- logovati svaki neuspešan pokušaj;
- posle maksimalnog broja pokušaja vratiti jasnu grešku korisniku;
- ne sme se vratiti broj koji nije uspešno rezervisan/upisan.

## Response kodovi

Preporučeni kodovi:

- `OK`
- `VALIDATION_ERROR`
- `PERMISSION_DENIED`
- `NOT_FOUND`
- `CONFLICT`
- `NUMBER_RESERVATION_FAILED`
- `SHAREPOINT_ERROR`
- `UNEXPECTED_ERROR`
