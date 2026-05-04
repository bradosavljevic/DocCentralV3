# Claude Code instrukcije

Ovaj folder sadrži uputstva za Claude Code.

## Redosled čitanja

1. `/README.md`
2. `/templates/claude-code-development-brief.md`
3. `/prompts/master-claude-code-prompt.md`
4. `/skills/*.md`
5. `/business/*.md`
6. `/architecture/*.md`
7. `/data-model/*.md`
8. `/power-apps/*.md`
9. `/power-automate/*.md`
10. `/security/*.md`
11. `/testing/test-matrix.md`

## Glavna pravila

- Ne praviti Dashboard.
- Ne praviti ekran Svi predmeti.
- Dokumenti se gledaju u SharePoint listama.
- Create/Edit/Delete kroz Power Automate.
- Delovodni broj mora biti concurrency-safe.
- Korisnici imaju SharePoint Read Only.
- Backend permissions su obavezne.
- Zaključana godina se ne može otključati.
- App Config je konfiguracioni layer.

## PACode pravilo

Svi fajlovi sa kodom koje Claude Code generiše moraju biti smešteni u root folder `PACode`.

Ne sme se generisati code fajl u root folder ili u dokumentacione foldere, osim ako korisnik izričito traži drugačije.

Primeri fajlova koji moraju ići u `PACode`:

- `.powerfx`
- `.json`
- `.ps1`
- `.sh`
- `.yaml`
- `.js`
- `.ts`
- `.md` fajlovi koji sadrže isključivo code/specifikaciju za izvršenje
- Power Automate definition/pseudocode fajlovi
- Power Apps formula fajlovi
