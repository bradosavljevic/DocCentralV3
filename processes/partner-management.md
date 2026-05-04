# Process: Partner Management

## Svrha

Partneri se koriste pri zavođenju dokumenata.

Partneri su posebna stavka u meniju, nisu deo administracije šifarnika.

## Dozvole

Korisnik koji zavodi dokument može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

## Brisanje

Kada se partner izbriše:

- ne sme se više prikazivati za izbor u novim dokumentima
- istorijski dokumenti u `Svi predmeti` moraju zadržati podatke o partneru
- brisanje ne sme pokvariti lookup, prikaz ili istoriju već unetih dokumenata

## Preporučeni tehnički model

Preporučuje se soft-delete umesto fizičkog brisanja.

Ako se radi fizičko brisanje, potrebno je pre toga obezbediti snapshot partner podataka na dokumentu.
