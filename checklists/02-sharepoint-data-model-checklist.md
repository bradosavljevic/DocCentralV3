# SharePoint Data Model Checklist

Za svaku listu ili biblioteku proveriti:

## Osnovno

- [ ] Naziv liste
- [ ] Interni naziv
- [ ] URL
- [ ] Svrha
- [ ] Broj itema ako je poznat
- [ ] Threshold rizik ako postoji

## Kolone

- [ ] Display name
- [ ] Internal name
- [ ] Tip kolone
- [ ] Required
- [ ] Unique
- [ ] Indexed
- [ ] Default value
- [ ] Validation formula
- [ ] Choice vrednosti
- [ ] Lookup veza
- [ ] Person or Group podešavanja

## Permissions

- [ ] Nasleđivanje prava
- [ ] Unique permissions
- [ ] Grupe
- [ ] Rola service account-a
- [ ] Read Only model za korisnike
- [ ] Direktan write pristup ako postoji

## Rizici

- [ ] Nedostaju indeksi za filtere
- [ ] Delegation problem
- [ ] Unique constraint nije podešen gde je potreban
- [ ] Lookup veza nije dokumentovana
- [ ] Person polja nisu jasno definisana
