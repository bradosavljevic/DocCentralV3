# Power Apps — App.OnStart / inicijalizacija

## Status

```text
DOPUNJENO NA OSNOVU POWER PLATFORM SOLUTION-A
```

## Činjenice

| Element | Vrednost |
|---|---|
| Verzija aplikacije | `g_varVersion = 2.1.155` |
| Trenutni korisnik | `g_varCurrentUser = Office365Users.MyProfileV2()` |
| Master kolekcija | `colMasterData` |
| Izvor | `EV_DocCentral21_lstAppConfig` |

## Uloga

App.OnStart / inicijalizacija pokriva:

- verziju aplikacije,
- korisnički kontekst,
- učitavanje App Config konfiguracije,
- formiranje šifarnika i kolekcija.

## Rizici

- Previše podataka na startu može usporiti aplikaciju.
- Kompleksan JSON u Canvas aplikaciji otežava održavanje.
- Security logika ne sme biti samo lokalna kolekcija.

## Preporuke

- App Config validirati backend logikom.
- Uvesti JSON schema za svaki App Config tip.
- Uvesti kontrolisan error handling ako konfiguracija nije validna.
