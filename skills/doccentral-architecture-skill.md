# Skill: DocCentral Architecture

## Kada koristiti

Koristi ovaj skill kada projektuješ ili menjaš arhitekturu DocCentral V3 rešenja.

## Pravila

- SharePoint je data/document layer.
- Canvas app je frontend.
- Power Automate je business/write layer.
- Korisnici imaju SharePoint Read Only.
- Service account izvršava write operacije.
- Backend permissions su obavezne.
- App Config je konfiguracioni layer.
- Entra grupe su konfigurabilne po klijentu.
- SAP import se ignoriše u ovoj fazi.

## Ne smeš

- hardkodovati tenant-specific grupe
- osloniti security samo na UI filtering
- direktno dozvoliti korisnički Write u SharePoint
- generisati delovodni broj u Canvas app bez server-side zaštite


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
