# Skill: SharePoint Data Model

## Pravila

- Koristiti konkretna imena lista i kolona kada su poznata.
- Ako nije poznato, označiti kao NEPOZNATO.
- Ne izmišljati interna imena.
- XML export koristiti za proveru schema.
- AppConfig.csv koristiti za šifarnike i konfiguraciju.
- Korisnici imaju Read Only.
- Service account ima RW.
- Item/folder/file permissions su obavezne.

## Glavne liste

- Svi predmeti
- Partneri
- App Config
- Rezervisani Brojevi
- Podsetnici
- Audit/Log
- konfiguracione/šifarnici liste

## Istorija partnera

Dokument mora čuvati istoriju partnera čak i ako se partner obriše ili deaktivira.


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
