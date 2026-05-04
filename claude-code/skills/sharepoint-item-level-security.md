# Skill: SharePoint Item-Level Security

## Cilj

Definisati pouzdan model za item/folder/file permissions.

## Standardni tok

1. Kreirati item/folder/file servisnim nalogom.
2. Break inheritance ako dokument nije globalno vidljiv.
3. Ukloniti inherited permissione ako je potrebno.
4. Dodeliti RW servisnom nalogu.
5. Dodeliti RW ownerima ako postoji owner model.
6. Dodeliti Read odgovarajućim grupama/organizacijama.
7. Logovati rezultat.
8. Ako permission assignment ne uspe, vratiti kontrolisanu grešku.

## Pravila

- Ne slati prazne email adrese u grant access akcije.
- Pre grant access validirati recipients array.
- Ako se koristi semicolon-separated string, ukloniti prazne vrednosti.
- Bolje je koristiti jasno formiran array i kontrolisan loop nego nevalidan string.

## Audit

Logovati:

- item/file ID;
- grupe kojima su prava dodeljena;
- role vrednosti;
- greške po pojedinačnoj grupi;
- correlationId.

## Performanse

Kod velikih biblioteka/liste:

- izbegavati nepotrebno break inheritance na svakom itemu ako nije poslovno potrebno;
- indeksirati security filter kolone;
- koristiti batching gde je moguće;
- dokumentovati limitacije SharePoint permission modela.
