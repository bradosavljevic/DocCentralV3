# Power Apps Canvas aplikacija - zahtevi

## Uloga aplikacije

Canvas aplikacija služi za:

- unos novog dokumenta
- izbor ili generisanje delovodnog broja
- prikaz dokumenata koje korisnik treba da odobri
- prikaz i upravljanje podsetnicima
- arhiviranje
- zaključenje godine
- administraciju šifarnika
- upravljanje partnerima

## Šta aplikacija ne sadrži

- nema Dashboard
- nema ekran "Svi predmeti"
- nema globalni pregled svih dokumenata
- dokumenti se gledaju direktno u SharePoint listama

## UX odluke

Claude Code može sam odlučiti:

- finalni broj ekrana
- nazive ekrana
- raspored kontrola
- vizuelni dizajn
- navigacioni layout

Uz obavezu da poštuje poslovna pravila iz dokumentacije.

## Obavezni ekrani / funkcionalne oblasti

Minimalno pokriti:

- Novi predmet
- Dokumenti za odobravanje
- Podsetnici
- Arhiviranje
- Zaključenje godine
- Administracija
- Partneri
- Rezervisani brojevi

## Tehnički standardi

- responsive design
- clean App Checker
- Formula-level error management
- IfError za Patch/Flow pozive
- gbl / loc / col naming
- delegabilne formule
- bez live search query-ja na svaki karakter
- bez referenciranja kontrola sa drugih ekrana
- bez nested galleries osim ako je opravdano
- App.Formulas gde ima smisla
- centralizovane named formulas za lookup/config/translation
- standard konektori osim ako korisnik izričito kaže drugačije

## Write operacije

Create/Edit/Delete ne smeju direktno raditi Patch u SharePoint ako korisnici imaju Read Only.

Sve takve operacije treba da idu kroz Power Automate flow-ove.

## Read operacije

Read operacije mogu koristiti:

- SharePoint
- Office 365 Users
- Microsoft 365 Groups
- App Config
- konfiguracione liste
