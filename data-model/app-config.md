# SharePoint lista: App Config

## Svrha

App Config sadrži šifarnike i konfiguracije koje aplikacija koristi u runtime-u.

Dostavljeni AppConfig.csv predstavlja osnovu za konačnu listu šifarnika i vrednosti.

## Tipovi konfiguracija

App Config može sadržati:

- tipove dokumenata
- statuse
- procesne konfiguracije
- delovodne knjige
- aktivnu godinu
- arhivske znakove
- konfiguracije podsetnika
- konfiguracije odobravanja
- organizacione jedinice
- tekstove i prevode
- vidljivost polja
- validaciona pravila
- konfiguraciju Entra grupa po klijentu

## Delovodne knjige

App Config / Delovodne knjige sadrži sledeći delovodni broj i godinu.

Pravila:

- aktivna godina se menja kod zaključenja godine
- zaključana godina se ne može ponovo otvoriti
- za novu godinu kreira se nova delovodna knjiga
- generisanje broja mora biti concurrency-safe

## ProcesConfig

ProcesConfig može definisati:

- statuse
- tokove statusa
- approval korake
- tipove dokumenata
- pravila vidljivosti
- obavezna polja
- korisnike/grupe za odobravanje

## Napomena

Claude Code mora koristiti App Config kao konfiguracioni layer i izbegavati hardkodovane vrednosti gde god je moguće.
