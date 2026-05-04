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
