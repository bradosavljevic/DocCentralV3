# Skill: Delovodni broj

## Kritičan zahtev

Dva dokumenta nikada ne smeju imati isti delovodni broj.

## Pravila

- DelovodniBroj je unikatan.
- Sledeći broj se čuva u Delovodnim knjigama / App Config.
- Rezervisani brojevi postoje u posebnoj listi.
- Generisanje mora biti concurrency-safe.
- Koristiti Power Automate za generisanje.
- Koristiti retry logiku.
- Koristiti audit log.
- Razmotriti ETag / optimistic locking.
- Unique constraint je obavezan kao zaštita.

## Reserved mode

Ako se koristi rezervisani broj:

- validiraj da postoji
- validiraj godinu
- zavedi dokument
- tek nakon uspeha obriši rezervisani broj

## Next mode

Ako se koristi sledeći broj:

- pročitaj brojač
- generiši broj
- pokušaj kreiranje dokumenta
- uvećaj brojač
- ako konflikt, retry
- ako neuspeh, loguj i vrati grešku


## Naming / postojeći objekti

Koristi solution `DocCentralV3`, connection references `CR_DocCentralV3_*`, environment variables `EV_DocCentralV3_*` i cloud flow nazive sa prefixom `CF_DocCentralV3_`. Ne koristiti prefix `PA_` za nove flow-ove. Svi zasebni code fajlovi moraju biti u folderu `PACode`.
