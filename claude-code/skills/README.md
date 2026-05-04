# Claude Code Skills - DocCentral / e-pisarnica

Ovaj folder sadrži obavezna pravila, obrasce i razvojne smernice koje Claude Code mora koristiti pri izradi nove verzije DocCentral rešenja.

## Izvor istine

Claude Code ne sme da izmišlja strukturu rešenja. Kao izvor istine koristi:

1. postojeću GitHub dokumentaciju projekta,
2. eksportovani Power Platform solution,
3. SharePoint XML/JSON/CSV eksportovane šeme,
4. App Config konfiguracije,
5. korisnički definisana pravila iz projekta.

Ako podatak nije potvrđen u dokumentaciji ili solution-u, mora biti označen kao `NEPOZNATO`.

## Obavezna arhitektonska pravila

- Canvas aplikacija je UI sloj, ne sme biti primarni write/security sloj.
- Korisnici imaju Read Only prava nad SharePoint-om.
- Create/Edit/Delete operacije idu preko Power Automate flow-ova.
- Power Automate radi pod servisnim nalogom.
- SharePoint item/file/folder permission model mora fizički ograničiti pristup podacima.
- UI filtriranje u Power Apps-u nije dovoljno kao security model.
- Delovodni broj je enterprise-critical funkcionalnost i mora biti concurrency-safe.
- `DelovodniBroj` mora ostati unikatan.
- SAP / eksterni import se za sada ignoriše, osim ako korisnik kasnije eksplicitno traži drugačije.

## Standard odgovora / dokumentacije

Svaki development brief, analiza ili tehnička odluka mora razdvajati:

- `ČINJENICE`
- `PRETPOSTAVKE`
- `NEPOZNATO`
- `PREPORUKE`
- `ODLUKE`

## Skill fajlovi

- `project-context.md` - poslovni i tehnički kontekst projekta
- `power-apps-canvas-development.md` - pravila za Canvas aplikaciju
- `power-apps-responsive-ui.md` - responzivni UI i UX pravila
- `power-apps-form-patterns.md` - unos, pregled, izmena i arhiviranje dokumenata
- `power-apps-localization.md` - srpski/engleski i lokalizacija
- `power-apps-security-filtering.md` - security filtering u aplikaciji
- `power-automate-backend-patterns.md` - backend flow arhitektura
- `power-automate-sharepoint-write-pattern.md` - SharePoint write preko flow-ova
- `power-automate-error-handling.md` - error handling i retry
- `power-automate-logging-audit.md` - centralni audit i logovanje
- `power-automate-child-flow-patterns.md` - child flow standardi
- `sharepoint-data-modeling.md` - SharePoint liste, biblioteke i kolone
- `sharepoint-permissions.md` - SharePoint security model
- `sharepoint-item-level-security.md` - item/folder/file permission breaking
- `sharepoint-document-libraries.md` - biblioteke dokumenata
- `delovodni-broj-concurrency.md` - concurrency-safe delovodni broj
- `app-config-driven-development.md` - App Config kao konfiguracioni sloj
- `document-lifecycle.md` - životni ciklus dokumenta
- `partner-management.md` - Partneri i ažuriranje podataka
- `approval-process.md` - proces odobravanja
- `archive-process.md` - arhiviranje i zaključavanje delovodnika
- `github-documentation-style.md` - stil GitHub dokumentacije
- `coding-standards.md` - standardi za kod i formule
- `testing-strategy.md` - test strategija
- `deployment-checklist.md` - deployment kontrolna lista
