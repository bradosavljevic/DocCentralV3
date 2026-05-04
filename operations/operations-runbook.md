# Operations Runbook

## Dnevne operacije

- Proveriti da li scheduled reminder flow radi.
- Proveriti log listu za greške.
- Proveriti neuspele flow run-ove.
- Proveriti da nema neuspešnih generisanja delovodnog broja.

## Mesečne operacije

- Pregled audit log-a.
- Pregled šifarnika.
- Provera neaktivnih partnera.
- Provera rezervisanih brojeva.

## Godišnje operacije

Pre zaključenja godine:

1. proveriti da su svi dokumenti statusa `Arhivirano`
2. proveriti da nema dokumenata u drugim statusima
3. proveriti da je lista rezervisanih brojeva prazna
4. generisati Arhivsku knjigu
5. zaključiti godinu
6. kreirati novu delovodnu knjigu za sledeću godinu
7. promeniti aktivnu godinu u App Config

## Incidenti

Ako dođe do problema sa delovodnim brojem:

- ne popravljati ručno bez analize
- proveriti audit log
- proveriti unique constraint
- proveriti flow run
- proveriti vrednost brojača u App Config
- proveriti da li postoje rezervisani brojevi
