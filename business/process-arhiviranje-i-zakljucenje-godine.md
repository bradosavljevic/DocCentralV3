# Proces: Arhiviranje i zaključenje godine

## Arhiviranje

Arhiviranje je posebna funkcionalnost i poseban ekran u aplikaciji.

Korisnik može da izlista sve dokumente koji su u statusu `Zavedeno` za tekuću godinu i da ih arhivira sa odgovarajućim arhivskim znacima.

## Direktan prelaz

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen.

Ovo je jedina direktna putanja arhiviranja.

## Arhivska knjiga

PDF generisanje ulazi u prvu verziju samo za Arhivsku knjigu.

## Zaključenje godine

Na kraju godine korisnik može da zaključi godinu.

Uslovi za zaključenje:

- svi dokumenti iz tekuće godine moraju biti u statusu `Arhivirano`
- ne sme postojati dokument u bilo kom drugom statusu
- lista rezervisanih brojeva mora biti prazna

Nakon zaključenja:

- kreira se nova delovodna knjiga za sledeću godinu
- aktivna godina se menja u App Config
- više nije moguće zavođenje dokumenata u zaključenoj godini
- zaključana godina nikada ne može biti ponovo otključana

## Administrator confirmation

Dodatna potvrda administratora nije potrebna.

## Logging

Obavezno logovati:

- arhiviranje
- neuspešno arhiviranje
- zaključenje godine
- neuspešno zaključenje godine
- generisanje PDF Arhivske knjige
