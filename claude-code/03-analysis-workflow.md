# Claude Code – Analysis Workflow

Ovaj workflow koristi se za analizu Power Platform solution-a i pratećih SharePoint JSON fajlova.

## Faza 1 – Inventory

Popiši sve pronađene artefakte:

- Power Apps aplikacije
- Power Automate flow-ove
- SharePoint liste
- SharePoint biblioteke
- Connection references
- Environment variables
- Security groups
- Custom connectors
- Dataverse tabele, ako postoje
- Rešenja i publisher informacije

Za svaki artefakt navedi:
- naziv,
- tip,
- lokaciju fajla,
- kratak opis,
- status analize.

## Faza 2 – Business overview

Izvuci:
- svrhu aplikacije,
- korisničke role,
- glavne poslovne procese,
- ulazne podatke,
- izlazne dokumente,
- statuse,
- odobravanja,
- arhiviranje,
- integracije.

## Faza 3 – Data model

Za svaku SharePoint listu i biblioteku dokumentuj:
- naziv,
- svrhu,
- kolone,
- tipove kolona,
- obaveznost,
- lookup veze,
- person/group polja,
- choice vrednosti,
- unique constraints,
- indekse ako su poznati,
- pravila validacije,
- permissions model.

## Faza 4 – Power Apps analiza

Analiziraj:
- ekrane,
- navigaciju,
- OnStart,
- App.Formulas,
- kolekcije,
- varijable,
- kontrole,
- forme,
- galerije,
- submit logiku,
- Patch logiku,
- flow pozive,
- validacije,
- error handling,
- loading/spinner logiku,
- responsive ponašanje.

## Faza 5 – Power Automate analiza

Za svaki flow dokumentuj:
- naziv,
- trigger,
- ulazne parametre,
- akcije,
- konektore,
- SharePoint operacije,
- approval logiku,
- grananja,
- error handling,
- retry policy,
- concurrency podešavanja,
- izlaz ka Power Apps,
- zavisnosti od service account-a.

## Faza 6 – Security model

Analiziraj:
- korisničke role,
- SharePoint prava,
- M365 grupe,
- service account,
- vlasništvo nad flow-ovima,
- ko može da čita,
- ko može da piše,
- ko može da odobrava,
- ko može da arhivira,
- gde postoji rizik od direktnog pristupa podacima.

## Faza 7 – Known issues i technical debt

Obavezno identifikuj:
- hardkodovane vrednosti,
- spore upite,
- delegation rizike,
- direktan Patch gde nije poželjan,
- nedostatak error loga,
- nedostatak audit loga,
- slabu retry logiku,
- nejasnu validaciju,
- permission rizike,
- concurrency rizike.

## Faza 8 – Enterprise recommendations

Predloži unapređenja:
- ALM,
- environment strategy,
- solution layering,
- connection references,
- environment variables,
- logging,
- monitoring,
- unique number generation,
- responsive UI,
- multilingual UI,
- test strategy,
- deployment checklist.
