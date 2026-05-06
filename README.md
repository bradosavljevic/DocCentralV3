# DocCentral Refactor

Ovaj repozitorijum je dokumentacioni i razvojni paket za refaktor postojeće Power Platform aplikacije **DocCentral / e-pisarnica**.

Cilj nije razvoj potpuno nove aplikacije od nule, već refaktor postojeće aplikacije uz očuvanje poslovne logike, unapređenje kvaliteta koda, performansi, stabilnosti, responzivnosti, sigurnosti i održavanja.

## Glavni cilj

- Refaktorisati postojeću Canvas Power Apps aplikaciju.
- Zadržati postojeću poslovnu logiku koliko god je moguće.
- Rešiti postojeće probleme:
  - race condition kod dodele delovodnog broja
  - direktne SharePoint write operacije iz Canvas app
  - nedovoljno standardizovan error handling
  - veliki ekrani i tehnički dug
  - AppChecker nalaze
  - performanse kod velikih SharePoint lista
- Uvesti jasan human-in-the-loop proces za Power Automate flow-ove i SharePoint strukturu.

## Ciljna arhitektura

```text
Power Apps Canvas App
    |
    | read: SharePoint / Power Automate read flows
    | write: Power Automate only
    |
Power Automate
    |
    | service account connection
    |
SharePoint Online
```

## Najvažnija pravila

1. Korisnici imaju Read prava nad SharePoint listama i bibliotekama.
2. Sve Create/Update/Delete operacije idu preko Power Automate flow-ova.
3. Power Automate write operacije rade preko servisnog naloga.
4. `AppConfig` ostaje centralni JSON configuration store.
5. `Svi predmeti` je glavna lista za metadata dokumenata.
6. `Dokumenta` je glavna biblioteka za fizičke dokumente i priloge.
7. Delovodni brojevi se ne smeju duplirati i ne smeju se preskakati.
8. MCP Canvas / Claude Code ne kreira flow-ove direktno, već piše implementacione vodiče.

## Novi solution context

Novi solution za razvoj koristi objekte iz screenshot-a:

- Canvas app: `DocCentralV3`
- Connection references:
  - `CR_DocCentralV3_Excel`
  - `CR_DocCentralV3_Office365Groups`
  - `CR_DocCentralV3_Office365Users`
  - `CR_DocCentralV3_OneDrive`
  - `CR_DocCentralV3_Outlook`
  - `CR_DocCentralV3_SharePoint`
- Environment variables:
  - `EV_DocCentralV3_docDokumenti`
  - `EV_DocCentralV3_docEmailDocs`
  - `EV_DocCentralV3_docExports`
  - `EV_DocCentralV3_lstAppConfig`
  - `EV_DocCentralV3_lstAuditLog`
  - `EV_DocCentralV3_lstPartneri`
  - `EV_DocCentralV3_lstPodsetnici`
  - `EV_DocCentralV3_lstRezervisaniBrojevi`
  - `EV_DocCentralV3_lstSviPredmeti`
  - `EV_DocCentralV3_SharePointSite`

## Kako koristiti

1. Pregledati `CLAUDE.md`.
2. Pregledati dokumentaciju u `docs/`.
3. Ručno implementirati potrebne SharePoint liste/kolone iz `sharepoint-guides/`.
4. Ručno implementirati Power Automate flow-ove iz `flow-guides/`.
5. Vratiti Claude Code-u tačne nazive, input/output schema i potvrdu da je komponenta napravljena.
6. Tek nakon toga menjati Canvas app formule i povezivanja.


## Development environment links

Power Apps Studio / edit link:

```text
https://make.powerapps.com/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/canvas/?action=edit&app-id=%2Fproviders%2FMicrosoft.PowerApps%2Fapps%2Fd17711d4-1417-4b8d-ac63-df49131264d0&solution-id=a2ca7e7f-8547-f111-bec6-0022489d3551
```

Runtime / play link:

```text
https://apps.powerapps.com/play/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/a/d17711d4-1417-4b8d-ac63-df49131264d0?tenantId=b5f62db0-39d2-4763-9c01-9f250adb3b41&hint=978a7273-8cff-45bb-b033-c4040a8a39dd&sourcetime=1778059741251
```

Detailed environment notes are in `docs/19-development-environment.md`.
