# 05 — Power Apps Analysis

## Status

```text
DOPUNJENO NA OSNOVU POWER PLATFORM SOLUTION-A
```

## Potvrđeno

- 1 Canvas aplikacija.
- 16 ekrana.
- App version: `g_varVersion = 2.1.155`.
- Current user: `g_varCurrentUser = Office365Users.MyProfileV2()`.
- `colMasterData` se puni iz `EV_DocCentral21_lstAppConfig`.

## Ekrani

- `scrHome`
- `scrAddDocument`
- `scrAddReminders`
- `scrReserveNumber`
- `scrDocSwap`
- `scrArchive`
- `scrCodes`
- `scrArchiveBook`
- `scrArchiveBookItems`
- `scrLockYear`
- `scrSettings`
- `scrPartneri`
- `scrEDocuments`
- `scrEmailDocs`
- `scrEmail`
- `scrProcess`

## Kolekcije

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

## Pravilo za novu verziju

Canvas app je UI sloj. Kritične operacije moraju ići preko Power Automate/backend sloja.

Canvas app ne sme sama da garantuje delovodni broj, security ili finalne Create/Edit/Delete transakcije.
