# Process: Arhiviranje i zaključenje godine

## Arhiviranje

Arhiviranje je posebna i bitna funkcionalnost aplikacije.

Korisnik može da izlista sve dokumente koji su u statusu `Zavedeno` za tekuću godinu i da ih arhivira sa odgovarajućim arhivskim znacima.

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen.

To je jedini direktan put arhiviranja.

## Zaključenje godine

Godina se može zaključiti samo ako:

- svi dokumenti iz tekuće godine imaju status `Arhivirano`
- ne postoji dokument u bilo kom drugom statusu
- lista rezervisanih brojeva za tu godinu je prazna

Kada se godina zaključi:

- kreira se nova delovodna knjiga za sledeću godinu
- menja se aktivna godina u App Config
- više nije moguće zavođenje dokumenata u zaključenoj godini

## Nepovratnost

Zaključana godina se nikada ne može ponovo otključati.

Ne sme postojati UI, flow, admin opcija ili backend akcija koja može ponovo otvoriti zaključanu godinu.

## Logging

Obavezno logovati:

- arhiviranje
- zaključenje godine
- neuspešno zaključenje godine
- pokušaj zavođenja u zaključanoj godini
