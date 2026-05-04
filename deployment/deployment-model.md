# Deployment model

## Model

Deployment se radi ručnim importom solution-a po tenant-u.

## Nema migracije

Migracioni plan nije potreban.

Novi klijenti kreću iz praznog okruženja.

Postojeći klijenti već koriste postojeće liste.

## Pre deployment checklist

- [ ] Kreiran SharePoint site
- [ ] Kreirane SharePoint liste
- [ ] Kreirane biblioteke/folder strukture
- [ ] Kreiran service account
- [ ] Service account ima potrebna RW prava
- [ ] Krajnji korisnici imaju Read Only
- [ ] Importovan solution
- [ ] Podešene connection references
- [ ] Podešene environment variables
- [ ] Popunjen App Config
- [ ] Popunjeni šifarnici
- [ ] Podešen Entra group mapping
- [ ] Testiran unos dokumenta
- [ ] Testirano odobravanje
- [ ] Testirano arhiviranje
- [ ] Testirano zaključenje godine
