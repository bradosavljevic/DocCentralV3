# Project Context – DocCentral / e-pisarnica

## Svrha projekta

DocCentral / e-pisarnica je poslovno rešenje zasnovano na Microsoft 365 tehnologijama, sa fokusom na evidentiranje, obradu, odobravanje i arhiviranje poslovnih dokumenata.

Primarni tehnološki elementi:
- Power Apps Canvas aplikacija
- Power Automate flow-ovi
- SharePoint Online liste i biblioteke
- Microsoft 365 korisnici i grupe
- Potencijalne integracije sa eksternim sistemima kao što su SAP, SEF ili Dynamics 365 Business Central

## Cilj dokumentacije

Cilj ovog GitHub projekta je da dokumentacija bude dovoljno jasna da može služiti kao:
- tehnička dokumentacija postojećeg rešenja,
- business i architecture overview,
- osnova za onboarding novog developera,
- input za Claude Code,
- input za razvoj unapređene enterprise verzije.

## Ključni poslovni zahtev

Sistem mora omogućiti da više korisnika istovremeno zavodi dokumenta, ali nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovo je enterprise zahtev i mora se tretirati kroz:
- concurrency-safe generisanje delovodnog broja,
- atomic ili optimistic locking pristup,
- retry logiku,
- unique constraint,
- audit log,
- error log,
- jasan rollback ili compensation scenario.

## Važno ograničenje

Dokumentacija se mora zasnivati na stvarnim artefaktima:
- exportovan Power Platform solution ZIP,
- JSON konfiguracije SharePoint lista,
- JSON konfiguracije biblioteka,
- exportovani Power Automate flow definicije,
- Power Apps YAML / source fajlovi ako postoje,
- ručno dostavljeni opisi procesa.

Bez tih fajlova, zaključak mora biti označen kao `NEPOZNATO`.
