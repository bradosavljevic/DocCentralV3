# Claude Code Development Brief - DocCentral V3

## Uloga

Ponašaj se kao senior Power Platform arhitekta i developer.

Projektuješ novu verziju DocCentral V3 / e-Pisarnica rešenja na osnovu dokumentacije u ovom repository-ju.

## Primarni cilj

Napraviti enterprise-ready Power Platform rešenje za:

- zavođenje dokumenata
- jedinstven delovodni broj
- rezervisane brojeve
- odobravanje
- podsetnike
- arhiviranje
- zaključenje godine
- partnere
- administraciju šifarnika
- audit/logging
- PDF Arhivsku knjigu

## Ne smeš prekršiti

- Korisnici nemaju direktan Write u SharePoint.
- Create/Edit/Delete ide kroz Power Automate.
- Delovodni broj mora biti concurrency-safe.
- Dva dokumenta nikada ne smeju dobiti isti delovodni broj.
- Zaključana godina se nikada ne može otključati.
- Backend permissions su obavezni.
- UI filtering nije dovoljan security model.
- Partner history mora ostati na dokumentima.
- SAP import se ignoriše u ovoj fazi.

## Možeš sam odlučiti

- finalni UX
- finalni broj ekrana
- vizuelni dizajn
- nazive ekrana
- raspored kontrola

## Obavezno koristi

- README.md
- business/*.md
- architecture/*.md
- data-model/*.md
- power-apps/*.md
- power-automate/*.md
- security/*.md
- testing/*.md
- checklists/*.md
- skills/*.md

## Output koji treba da proizvedeš

- predlog strukture Canvas app
- predlog SharePoint lista
- predlog Power Automate flow-ova
- detaljne formule/pseudocode gde je potrebno
- test matricu
- deployment instrukcije
- gap listu ako nešto nije dovoljno definisano
