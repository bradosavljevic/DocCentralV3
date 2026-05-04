# Skill: Document Lifecycle

## Cilj

Definisati standardni životni ciklus dokumenta u DocCentral rešenju.

## Tipični statusi

Statusi moraju biti potvrđeni iz dokumentacije/App Config-a. Ako nisu potvrđeni, koristiti kao preporuku:

- Nacrt
- U obradi
- Zavedeno
- Na odobrenju
- Odobreno
- Odbijeno
- Vraćeno na doradu
- Arhivirano

Poznato je da su `Zavedeno` i `Arhivirano` važni statusi.

## Standardni tok

1. Kreiranje predmeta/dokumenta.
2. Unos metapodataka.
3. Dodavanje fajlova/priloga.
4. Validacija.
5. Zavođenje i dodela delovodnog broja.
6. Dodela prava pristupa.
7. Odobravanje ako je potrebno.
8. Arhiviranje.
9. Zaključavanje nakon arhiviranja.

## Pravila

- Dokument u statusu `Arhivirano` ne sme se menjati osim kroz posebno definisanu administratorsku proceduru.
- Pre prelaska u `Zavedeno`, mora postojati potvrđen delovodni broj.
- Pre arhiviranja proveriti da li su ispunjeni poslovni uslovi.
- Svaka promena statusa mora biti auditovana.

## Nepoznato

- Kompletna status matrica.
- Ko sme da izvrši svaku statusnu promenu.
- Da li je approval obavezan za sve tipove dokumenata.
- Da li postoje SLA rokovi.
