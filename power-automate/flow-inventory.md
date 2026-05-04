# Power Automate — Flow Inventory

## Status

```text
DOPUNJENO NA OSNOVU POWER PLATFORM SOLUTION-A
```

## Činjenice

Solution sadrži 20 flow-ova.

- `CF_DocCentral21_AddPartner`
- `CF_DocCentral21_UploadDoc_V4`
- `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`
- `CF_DocCentral21_GetPartners`
- `CF_DocCentral21_ResetDelovodnika`
- `CF_DocCentral21_ProcessFlow`
- `CF_DocCentral21_EFaktureProces`
- `CF_DocCentral21_EmailDoc`
- `CF_DocCentral21_GetFileEmail`
- `CF_DocCentral21_AdHocApproval`
- `CF_DocCentral_HTMLTabela`
- `CF_DocCentral21_LockYear`
- `CF_DocCentral21_GetLockYearData`
- `CF_DocCentral21_ExportCodes`
- `CF_DocCentral21_ArchiveDocumentsV2`
- `CF_DocCentral21_CheckArchiveBookData`
- `CF_DocCentral21_DocSwap`
- `CF_DocCentral21_ReserveNumber`
- `CF_DocCentral21_UploadDoc_V2`
- `CF_DocCentral21_AddReminder`

## Kritični flow-ovi za delovodni broj

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
CF_DocCentral21_ReserveNumber
CF_DocCentral21_ResetDelovodnika
CF_DocCentral21_LockYear
CF_DocCentral21_GetLockYearData
```

## Kritični flow-ovi za dokumente

```text
CF_DocCentral21_UploadDoc_V4
CF_DocCentral21_UploadDoc_V2
CF_DocCentral21_DocSwap
CF_DocCentral21_ArchiveDocumentsV2
```

## Email intake

```text
CF_DocCentral21_EmailDoc
CF_DocCentral21_GetFileEmail
```

## Partneri

```text
CF_DocCentral21_AddPartner
CF_DocCentral21_GetPartners
```

## Nepoznato

Za svaki flow još treba dokumentovati trigger, akcije, input/output schema, konektore, retry, error handling, owner, run-only user i service account.

## Preporuka

Napraviti poseban `.md` fajl po svakom flow-u u sledećoj fazi.
