# Skill: SharePoint Data Modeling

## Cilj

SharePoint model mora biti dokumentovan i projektovan tako da podrži poslovne procese, bez duplih poslovnih identifikatora i bez zavisnosti od samo UI logike.

## Poznate liste / biblioteke

- `Svi predmeti`
- `Partneri`
- `App Config`
- `Rezervisani Brojevi`
- dokument biblioteke
- pomoćne liste ako su potvrđene u eksportu

## Pravila za kolone

Za svaku listu dokumentovati:

- display name;
- internal name;
- tip kolone;
- da li je required;
- da li je indexed;
- da li je unique;
- default vrednost;
- choice vrednosti;
- lookup relacije;
- calculated/hidden/system upotrebu;
- da li je koristi Canvas app;
- da li je koristi Power Automate.

Ako podatak nije poznat, označiti kao `NEPOZNATO`.

## Unique constraints

- `DelovodniBroj` mora biti unique gde je poslovno glavni identifikator.
- Title ne sme automatski biti tretiran kao jedini poslovni ključ ako dokumentacija to ne potvrđuje.

## Indeksi

Indeksirati kolone koje se koriste za:

- filtriranje po statusu;
- filtriranje po organizacionoj jedinici;
- filtriranje po delovodnom broju;
- lookup partnera;
- datumske opsege;
- arhivu/godinu;
- aktivne zapise.

## Zabranjeno

- Pretpostaviti internal name ako postoji rizik da nije isti kao display name.
- Rekonstruisati listu bez provere XML/JSON exporta.
- Koristiti neindeksirane filtere za velike liste bez napomene o riziku.
