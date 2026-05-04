# Skill: App Config Driven Development

## Cilj

`App Config` je konfiguracioni sloj za šifarnike, pravila i podešavanja aplikacije.

## Činjenice

- `App Config` sadrži šifarnike i konfiguracije.
- JSON vrednosti iz `App Config` aplikacija koristi za konfiguraciju i šifarnike.
- `Delovodne knjige` su deo konfiguracije.

## Pravila

- Ne hardkodovati šifarnike ako postoje u `App Config`.
- Učitavati samo aktivne konfiguracije ako postoji polje/status za aktivnost.
- Parsirati JSON kontrolisano.
- Ako JSON schema nije poznata, dokumentovati kao `NEPOZNATO`.
- Ako konfiguracija upravlja UI ponašanjem, pravilo mora biti dokumentovano.

## Kategorije konfiguracije

Moguće kategorije, potvrditi iz exporta:

- tipovi dokumenata;
- delovodne knjige;
- statusi;
- organizacione jedinice;
- vidljivost polja;
- approval pravila;
- prevodi;
- role/security mapping;
- document lifecycle pravila.

## Power Apps upotreba

- Na startu učitati konfiguracije.
- Kreirati kolekcije za šifarnike.
- Ne oslanjati se na staru konfiguraciju ako je operacija kritična.

## Power Automate upotreba

- Flow mora sam validirati konfiguraciju za kritične operacije.
- Za delovodni broj flow ne sme verovati samo vrednosti koju je poslala Canvas aplikacija.
