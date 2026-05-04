# Claude Code development brief

## Cilj

Razviti novu verziju DocCentralV3 Power Platform rešenja na osnovu ove dokumentacije.

## Najvažnije pravilo

Sav kod koji Claude Code generiše mora biti smešten u:

```text
PACode
```

## Rešenje

- Solution: `DocCentralV3`
- Canvas App: `DocCentralV3`
- Publisher: `GoProDocCentral`
- Version: `3.0.0.0`

## Arhitektonski obrazac

- Canvas App je UI
- Power Automate je backend/write sloj
- SharePoint je data/document storage
- App Config je izvor konfiguracije
- Entra grupe su konfiguracione
- Service account izvršava write operacije
- Korisnici imaju Read Only prava

## UX pravila

- Nema dashboard-a
- Nema pregleda svih dokumenata u aplikaciji
- Dokumenti se gledaju direktno u SharePoint-u
- Aplikacija mora imati:
  - Novi predmet
  - Dokumenti za odobrenje
  - Podsetnici
  - Arhiviranje
  - Zaključenje godine
  - Administracija
  - Partneri

## Dokumenti

- Ne kreirati foldere
- Svi fajlovi idu u root biblioteke
- Glavni dokument i prilozi se razlikuju metapodacima
- Svaki fajl mora imati unikatan sistemski naziv
- Originalno ime fajla se čuva kao metapodatak

## Flow pravila

Koristiti flow-ove sa `CF_` prefiksom.

Glavni flow:

```text
CF_DocCentralV3_CreateDocument
```

## Kvalitet

Poštovati:

- responsive design
- čist App Checker
- formula-level error management
- IfError za Patch/Flow pozive
- gbl/loc/col naming
- delegabilne formule
- bez live search query-ja na svaki karakter
- bez cross-screen control references
- bez nested galleries osim ako je opravdano
