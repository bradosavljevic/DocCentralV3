# Skill: GitHub Documentation Style

## Cilj

Sva dokumentacija mora biti pogodna za GitHub repozitorijum i budući razvoj kroz Claude Code.

## Struktura

Preporučena struktura:

```text
docs/
architecture/
data-model/
power-apps/
power-automate/
configuration/
known-issues/
claude-code/
claude-code/skills/
templates/
checklists/
prompts/
backlog/
```

## Stil pisanja

Svaki dokument treba da ima:

- naslov;
- svrhu;
- činjenice;
- pretpostavke;
- nepoznato;
- preporuke;
- tehničke odluke;
- otvorena pitanja ako postoje.

## Pravila

- Ne koristiti marketinški narativ.
- Ne izmišljati nazive.
- Koristiti konkretne nazive iz solution-a kada postoje.
- Za svaki nedostajući podatak napisati `NEPOZNATO`.
- Koristiti tabele za kolone, flow-ove, ekrane i konfiguracije.
- Koristiti checklist format za implementacione zadatke.

## Naming convention fajlova

- mala slova;
- reči odvojene crticom;
- `.md` ekstenzija;
- bez razmaka i srpskih karaktera u nazivima fajlova.

Primer:

```text
delovodni-broj-concurrency.md
power-automate-error-handling.md
sharepoint-permissions.md
```
