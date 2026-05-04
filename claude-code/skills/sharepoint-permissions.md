# Skill: SharePoint Permissions

## Cilj

Security model mora biti stvaran SharePoint backend security, ne samo UI filtriranje.

## Činjenice

- Krajnji korisnici imaju Read Only prava.
- Servisni nalog ima RW prava.
- Owneri imaju RW tamo gde je poslovno dozvoljeno.
- Members/Viewers imaju Read.
- Admin grupa nije potvrđena kao postojeća.
- Pristup se zasniva na organizacionim jedinicama i grupama.

## Pravila

- Nakon kreiranja itema/fajla primeniti odgovarajuća prava.
- Ako dokument nije za sve korisnike, koristiti break inheritance.
- Prava moraju biti primenjena na item/folder/file, zavisno od modela biblioteke.
- Canvas aplikacija sme dodatno filtrirati, ali backend mora sprečiti pristup.

## Dokumentovati

Za svaku listu/biblioteku:

- ko ima Read;
- ko ima RW;
- da li se koristi inheritance;
- da li se koristi item-level permission;
- koje grupe se dodeljuju;
- da li servisni nalog ima dovoljan pristup.

## Rizici

- Previše item-level permission break-ova može uticati na performanse i administraciju.
- Nevalidne email adrese ili prazni recipienti mogu rušiti Power Automate akcije za grant access.
- Security ne sme zavisiti samo od grupe učitane u Canvas kolekciji.
