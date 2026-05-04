# Power Automate — Connection References

## Status

```text
DOPUNJENO NA OSNOVU POWER PLATFORM SOLUTION-A
```

## Činjenice

Solution sadrži 5 connection references.

- `gpdoccen_CF_DocumentCentral_SharedMailbox`
- `new_sharedexcelonlinebusiness_9b86f`
- `new_sharedonedriveforbusiness_49131`
- `pbmlv2_CR_CR_DocCentral21_Office365Outlook`
- `pbmlv2_CR_DocCentral21_SharePoint`

## Rizici

- Neispravno mapiranje connection references može oboriti import solution-a.
- Lične konekcije umesto service account konekcija su operativni rizik.
- Ako korisnici imaju Read Only, flow connection mora imati RW prava.

## Preporuke

- Dokumentovati vlasnika svake konekcije.
- Definisati service account za produkciju.
- Pripremiti deployment settings JSON.
