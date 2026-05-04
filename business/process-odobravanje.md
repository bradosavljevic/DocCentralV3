# Proces: Odobravanje dokumenta

## Cilj

Omogućiti kontrolisano odobravanje dokumenta, uz promenu statusa dokumenta kroz polje `Stanje`.

## Osnovna pravila

- Dokument ide na odobravanje jednom korisniku u jednom koraku.
- Ako ima više korisnika, proces je sekvencijalan.
- Postoji opcija slanja na grupu.
- Kada se šalje na grupu, više korisnika dobija obaveštenje.
- Za dati approval status prvi korisnik koji odobri završava taj korak.
- Odobrenje menja status dokumenta promenom polja `Stanje`.

## Vidljivost

U aplikaciji korisnik vidi dokumente koje nije odobrio.

U SharePoint listi korisnik vidi:

- dokumente koji su za njega
- dokumente gde je on učestvovao u procesu odobravanja
- dokumente gde je njegova grupa učestvovala u procesu odobravanja

## Statusi

Osnovni statusi:

- Zavedeno
- U odobravanju
- Odobreno
- Odbijeno
- Arhivirano

## Odbijanje

Ako korisnik odbije dokument:

- dokument dobija status `Odbijeno`
- dokument se vraća inicijatoru
- inicijator može izmeniti dokument ili metadata
- inicijator može pokrenuti novi proces odobravanja

## Napomena za implementaciju

Claude Code treba da projektuje proces tako da bude konfigurabilan kroz ProcesConfig, gde je moguće definisati statusne tokove i korisnike/grupe po koracima.
