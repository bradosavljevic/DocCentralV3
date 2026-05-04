# Skill: Deployment Checklist

## Cilj

Definisati kontrolnu listu za pripremu, import i validaciju DocCentral rešenja.

## Pre deployment

- Proveriti solution package.
- Proveriti connection references.
- Proveriti environment variables.
- Proveriti servisni nalog.
- Proveriti SharePoint site i liste.
- Proveriti biblioteke dokumenata.
- Proveriti App Config aktivne vrednosti.
- Proveriti unique constraints.
- Proveriti indekse.
- Proveriti permissions.

## Import

- Importovati solution u ciljano okruženje.
- Povezati connection references.
- Podesiti environment variables.
- Uključiti flow-ove kontrolisano.
- Proveriti owner/run-only korisnike.

## Post deployment

- Testirati Canvas aplikaciju.
- Testirati učitavanje konfiguracije.
- Testirati kreiranje predmeta.
- Testirati zavođenje i delovodni broj.
- Testirati upload dokumenta.
- Testirati permission breaking.
- Testirati arhiviranje.
- Testirati error handling.
- Proveriti audit log.

## Go-live

- Očistiti test podatke ako je potrebno.
- Resetovati brojače samo kroz dogovorenu proceduru.
- Zaključati konfiguracije koje ne smeju da se menjaju.
- Potvrditi servisni nalog i licence.
- Dokumentovati verziju solution-a.

## Rollback

Mora postojati plan za:

- vraćanje prethodne verzije solution-a;
- deaktivaciju novih flow-ova;
- vraćanje App Config vrednosti;
- očuvanje već kreiranih delovodnih brojeva;
- očuvanje audit loga.
