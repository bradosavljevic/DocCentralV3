# Security i permission model

## Cilj

Obezbediti da korisnici vide samo dokumente za koje imaju pravo pristupa i da ne mogu direktno menjati SharePoint podatke mimo kontrolisanih flow-ova.

## Osnovna pravila

- Korisnici imaju SharePoint Read Only.
- Sve Create/Edit/Delete operacije idu kroz Power Automate.
- Power Automate radi pod servisnim nalogom.
- Service account ima RW.
- Owners imaju RW.
- Members i Viewers imaju Read.
- Ne postoji posebna admin grupa.
- Prava zavise od organizacionih jedinica.
- Entra group mapping je konfigurabilan po klijentu.

## Backend security

Pristup se ne sme oslanjati samo na Power Apps UI filtriranje.

Mora se primeniti backend security:

- item-level permissions
- folder/file permissions
- break inheritance
- dodela prava korisnicima/grupama

## Canvas app security

Canvas app dodatno može:

- sakriti kontrole
- filtrirati prikaze
- proveriti role
- sprečiti pokretanje akcija za korisnika bez prava

Ali ovo je samo dodatni sloj, ne glavni security model.

## SharePoint permissions

Kod kreiranja dokumenta:

1. prekida se nasleđivanje prava
2. servisni nalog dobija RW
3. owner dobija RW
4. relevantni korisnici/grupe dobijaju Read
5. dokument/folder/file postaje vidljiv samo relevantnim osobama

## Entra grupe

Mapa Entra grupa zavisi od klijenta i mora biti u konfiguraciji.

Ne hardkodovati tenant-specific grupe u aplikaciji ili flow-ovima.
