# Environment variables - DocCentral V3

## Svrha

Ovaj dokument definiše environment variables koje su već kreirane u solution-u `DocCentralV3` i koje Claude Code mora koristiti u svim Power Apps i Power Automate predlozima.

## Pravila

- Ne hardkodovati SharePoint URL, list GUID, library GUID ili nazive lista u formulama/flow-ovima ako postoji environment variable.
- Koristiti postojeće display name i logical/name vrednosti.
- Pre svakog testiranja proveriti da svaka environment variable ima popunjenu `Current value` vrednost.
- `EV_DocCentralV3_SharePointSite` je osnovna promenljiva za SharePoint site.
- Liste i biblioteke se referenciraju kroz svoje environment variables.

## Lista environment variables

| Display name | Logical/name | Namena |
|---|---|---|
| `EV_DocCentralV3_SharePointSite` | `gpdoccen_EV_DocCentralV3_SharePointSite` | SharePoint site |
| `EV_DocCentralV3_lstSviPredmeti` | `gpdoccen_EV_DocCentralV3_lstSviPredmeti` | Glavna lista predmeta/dokumenata |
| `EV_DocCentralV3_lstPartneri` | `gpdoccen_EV_DocCentralV3_lstPartneri` | Lista partnera |
| `EV_DocCentralV3_lstAppConfig` | `gpdoccen_EV_DocCentralV3_lstAppConfig` | App Config / šifarnici / procesna konfiguracija |
| `EV_DocCentralV3_lstRezervisaniBrojevi` | `gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi` | Rezervisani delovodni brojevi |
| `EV_DocCentralV3_lstPodsetnici` | `gpdoccen_EV_DocCentralV3_lstPodsetnici` | Podsetnici |
| `EV_DocCentralV3_lstAuditLog` | `gpdoccen_EV_DocCentralV3_lstAuditLog` | Audit/log lista |
| `EV_DocCentralV3_docDokumenti` | `gpdoccen_EV_DocCentralV3_docDokumenti` | Dokument biblioteka za priloge/dokumente |
| `EV_DocCentralV3_docEmailDocs` | `gpdoccen_EV_DocCentralV3_docEmailDocs` | Biblioteka za email dokumente |
| `EV_DocCentralV3_docExports` | `gpdoccen_EV_DocCentralV3_docExports` | Biblioteka za export fajlove |

## Napomena

Flow-ovi se kreiraju kasnije, ali njihovi nazivi su već definisani u `power-automate/flow-inventory.md`.
