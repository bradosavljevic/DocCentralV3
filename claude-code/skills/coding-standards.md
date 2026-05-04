# Skill: Coding Standards

## Cilj

Definisati standarde za Power Apps formule, Power Automate flow-ove i JSON payload-e.

## Power Apps

- Koristiti jasna imena promenljivih.
- Globalne promenljive prefiksirati dosledno, npr. `gbl` ili `g_`.
- Lokalne promenljive prefiksirati dosledno, npr. `loc` ili `l_`.
- Kolekcije prefiksirati sa `col`.
- Named formule koristiti za helper logiku.
- Izbegavati dupliranje kompleksnih formula.
- Komentarima objasniti poslovno kritične delove.

## Power Automate

- Flow nazivi moraju biti jasni i domenski.
- Scope-ovi moraju imati standardna imena.
- Akcije moraju biti imenovane čitljivo, ne generički `Compose 1`, `Condition 2`.
- Koristiti `correlationId` kroz ceo flow.
- Svaki flow mora imati strukturisan output.

## JSON payload

- Koristiti stabilna imena polja.
- Ne slati nepotrebne podatke.
- Datume slati u ISO formatu ako je moguće.
- Brojeve slati kao brojeve, ne lokalizovane tekstove, osim kada je poslovno neophodno.

## Locale

- Formatiranje brojeva za prikaz raditi u Canvas aplikaciji.
- Backend treba da koristi stabilne numeričke vrednosti.
- Posebno paziti na srpski decimalni zarez.

## Zabranjeno

- Hardkodovani magic stringovi bez konfiguracije.
- Kritične poslovne vrednosti samo u label kontrolama.
- Flow logika bez error path-a.
- Silent failure.
