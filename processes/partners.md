# Proces: Partneri

## Opis

Partneri su zasebna stavka menija.

Lista partnera nije deo administracije.

## Ko sme da upravlja partnerima

Korisnik koji zavodi dokumente može:

- kreirati partnera
- menjati partnera
- brisati partnera
- pregledati partnera

## Brisanje partnera

Partner se može fizički obrisati ili učiniti nedostupnim za budući izbor, ali brisanje ne sme uticati na istorijske dokumente.

Dokumenti koji su već zavedeni moraju zadržati istorijske podatke o partneru.

## Pravilo za istoriju

Ako je partner izabran na dokumentu, u `Svi predmeti` treba čuvati dovoljno podataka da istorija ostane čitljiva i nakon brisanja partnera.

Preporuka:

- PartnerId
- PartnerNazivSnapshot
- PartnerPIBSnapshot
- PartnerMestoSnapshot
- PartnerAdresaSnapshot

## White-label pristup

Mapiranje partnera i dodatna polja mogu zavisiti od klijenta, ali osnovni proces mora ostati isti.
