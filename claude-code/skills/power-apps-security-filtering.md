# Skill: Power Apps Security Filtering

## Cilj

Definisati kako Canvas aplikacija treba da prikazuje podatke u skladu sa sigurnosnim pravilima.

## Činjenice

- Korisnici imaju Read Only prava nad SharePoint-om.
- Prava pristupa zavise od organizacionih jedinica i grupa.
- Backend SharePoint permission model mora stvarno ograničiti pristup itemima/fajlovima.
- Canvas aplikacija može dodatno filtrirati UI, ali to nije dovoljan security model.

## Pravila

- Na startu aplikacije učitati identitet korisnika.
- Učitati grupe korisnika preko Microsoft 365/Office 365/Graph logike dostupne u rešenju.
- Iz grupa odrediti organizacione jedinice koje korisnik sme da vidi.
- Prikaz liste predmeta filtrirati po dozvoljenim organizacionim jedinicama i backend dostupnosti.
- Akcije prikazati samo ako korisnik ima dozvolu za akciju.

## Važno

Ako SharePoint backend ne dozvoljava pristup itemu/fajlu, aplikacija ne sme pokušavati da zaobiđe taj model.

## Nepoznato

- Kompletna mapa grupa zavisi od klijenta.
- Pravila vidljivosti za role/grupe biće definisana kasnije.
- Ako ne postoji mapa grupa u dokumentaciji, ne izmišljati je.
