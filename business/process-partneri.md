# Proces: Partneri

## Cilj

Omogućiti korisnicima koji zavode dokumente da upravljaju partnerima.

## Funkcionalnosti

Korisnik koji zavodi dokumente može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

## Meni

Lista Partneri nije deo Administracije.

Partneri su zasebna stavka u meniju.

## Brisanje partnera

Partner se može fizički obrisati ili učiniti nedostupnim za budući izbor, ali brisanje ne sme uticati na istorijske dokumente.

Obavezno pravilo:

- ranije uneti dokumenti u listi Svi predmeti moraju zadržati istoriju partnera
- partner koji je obrisan ili deaktiviran ne sme se više birati za nove dokumente

## Preporuka za implementaciju

Enterprise-safe pristup je soft delete preko polja kao što su:

- IsActive
- DeletedOn
- DeletedBy

Ako se ipak koristi fizičko brisanje, dokument mora čuvati denormalizovane istorijske podatke partnera, kao što su:

- PartnerNaziv
- PartnerPIB
- PartnerMesto
- PartnerAdresa

## Logging

Logovati:

- neuspešno kreiranje partnera
- neuspešnu izmenu partnera
- neuspešno brisanje partnera
