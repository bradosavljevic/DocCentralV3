# Skill: Power Platform development standard

## Scope

Ovaj skill koristi Claude Code pri razvoju DocCentralV3.

## Kod

Sav generisani kod ide u:

```text
PACode
```

## Canvas naming

Koristiti prefikse:

- `gbl` za global variables
- `loc` za local/context variables
- `col` za kolekcije
- `cmp` za komponente
- `scr` za ekrane
- `gal` za galleries
- `frm` za forme
- `btn` za buttons
- `txt` za text input
- `cmb` za combobox
- `dp` za date picker

## Error handling

Koristiti:

- `IfError`
- jasne poruke korisniku
- standardizovan response iz flow-a
- audit log za backend greške

## Performance

- Delegabilne formule
- Bez pretrage na svaki karakter
- Search pokretati na dugme ili nakon debounce logike
- Bez cross-screen control references
- Bez nested galleries osim ako postoji opravdan razlog

## Security

- Ne oslanjati se samo na UI
- Backend permissions moraju biti primenjene
- Write operacije kroz flow
