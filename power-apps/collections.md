# Power Apps — Collections Inventory

## Status

```text
DOPUNJENO NA OSNOVU SOLUTION ANALIZE
```

## Identifikovane kolekcije

- `colMasterData`
- `colUsers`
- `colSettingsMenu`
- `colSettings`
- `colTranslations`
- `colLang`
- `colStanja`
- `colPartneri`
- `colAttachments`
- `colDelovodneKnjige`
- `colOrganizacioneJedinice`
- `colProcesConfig`
- `colTipoviDokumenta`
- `colValute`
- `colVrsteDokumenta`
- `colRezervisaniBrojevi`
- `colDocumentsSwap`
- `colArhiveDoc`
- `colEDocuments`
- `colEFakture`
- `colEmailDoc`

## Preporuke

- Kolekcije koristiti za UI i cache, ne kao izvor istine za kritične transakcije.
- `colDelovodneKnjige` i `colRezervisaniBrojevi` ne smeju garantovati unique broj bez backend potvrde.
- Za svaku kolekciju dokumentovati izvor, refresh trenutak, schema i ekrane koji je koriste.
