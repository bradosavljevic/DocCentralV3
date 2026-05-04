# Skill: Partner Management

## Cilj

Definisati pravila za listu `Partneri` i logiku kreiranja/ažuriranja partnera.

## Činjenice

- Partneri se proveravaju po PIB polju kada dolaze iz integracije/importa.
- Ako partner ne postoji, kreira se novi.
- Ako partner postoji, nova verzija treba da proveri promene u poljima kao što su naziv, grad i adresa.
- Ako su podaci promenjeni, partner treba automatski ažurirati po dogovorenim pravilima.
- SAP / eksterni import se za sada ignoriše u dokumentaciji razvoja, osim ako korisnik kasnije traži drugačije.

## Pravila

- PIB je poslovni identifikator za proveru partnera ako je potvrđen u modelu.
- Promene partnera moraju biti auditovane.
- Ne sme se kreirati duplikat partnera ako PIB već postoji.
- Ako je PIB prazan ili nevalidan, flow mora vratiti validacionu grešku ili koristiti posebno definisano pravilo.

## Polja za proveru

Potvrditi iz data modela, ali tipično:

- `PIB`
- `PoslovnoIme`
- `Mesto` / `Grad`
- `Adresa`
- `MB`

## Update strategija

1. Pronađi partnera po PIB-u.
2. Ako ne postoji, kreiraj partnera.
3. Ako postoji, uporedi relevantna polja.
4. Ako nema razlike, preskoči update.
5. Ako postoji razlika, update partnera.
6. Upiši audit šta je promenjeno.

## Nepoznato

- Da li postoji approval za promenu partnera.
- Da li korisnik mora potvrditi promene partnera.
- Da li se čuva istorija promena.
