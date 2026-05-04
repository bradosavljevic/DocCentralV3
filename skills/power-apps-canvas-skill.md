# Skill: Power Apps Canvas

## Standardi

- Responsive design
- Clean App Checker
- Formula-level error management
- IfError oko Patch/Flow poziva
- gbl/loc/col naming
- Delegabilne formule
- Bez live search query-ja na svaki karakter
- Bez cross-screen control references
- Bez nested galleries osim ako je opravdano
- Multi-language sr/en
- Serbian locale support

## App funkcionalnosti

Aplikacija treba da pokrije:

- Novi predmet
- Dokumenti za odobravanje
- Podsetnici
- Arhiviranje
- Zaključenje godine
- Administracija
- Partneri
- Rezervisani brojevi

## Ne praviti

- Dashboard
- Svi predmeti ekran
- globalni pregled svih dokumenata u aplikaciji

## Write operacije

Create/Edit/Delete preko Power Automate.

Ne direktan Patch za poslovne write operacije krajnjih korisnika.


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
