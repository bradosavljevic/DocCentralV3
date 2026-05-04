# Skill: Testing Strategy

## Cilj

Definisati minimalnu test strategiju za novu verziju DocCentral-a.

## Test oblasti

- Canvas UI testovi.
- Power Automate flow testovi.
- SharePoint data model testovi.
- Permission/security testovi.
- Delovodni broj concurrency testovi.
- App Config parsing testovi.
- Arhiviranje.
- Approval ako je implementiran.

## Kritični test: delovodni broj

Obavezno testirati:

- dva korisnika istovremeno klikću `Zavedi`;
- više paralelnih zahteva za istu godinu/knjigu;
- konflikt unique constraint-a;
- retry nakon konflikta;
- audit neuspelih pokušaja;
- povratni response u Canvas aplikaciju;
- da nijedan dokument ne dobije isti broj.

## Security testovi

- Korisnik iz jedne organizacione jedinice ne vidi tuđe dokumente.
- Direktan SharePoint pristup ne otkriva dokumente bez prava.
- Canvas UI ne prikazuje akcije bez dozvole.
- Backend flow odbija operaciju bez prava, čak i ako UI pokuša da pošalje zahtev.

## Flow testovi

Za svaki flow testirati:

- validan input;
- nedostajuće obavezno polje;
- nepostojeći item;
- permission denied;
- SharePoint grešku;
- retry scenario;
- response format.

## Dokumentovati rezultate

Svaki test scenario treba da ima:

- naziv;
- preduslove;
- korake;
- očekivani rezultat;
- stvarni rezultat;
- status;
- napomene.
