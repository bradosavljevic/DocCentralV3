# SharePoint lista: App Config

## Environment variable

```text
EV_DocCentralV3_lstAppConfig
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstAppConfig
```

## Namena

`App Config` sadrži šifarnike i konfiguracije koje aplikacija koristi.

## Primeri konfiguracija

- delovodne knjige
- aktivna godina
- tipovi dokumenata
- procesne konfiguracije
- statusi
- arhivski znaci
- vreme slanja podsetnika
- konfiguracija odobravanja
- konfiguracija rola i grupa
- konfiguracija vidljivosti polja
- lokalizacija ako postoji

## Važno

Konačna lista šifarnika i vrednosti data je kroz `AppConfig.csv`.

Claude Code mora tretirati App Config kao izvor konfiguracije, ne kao hardcoded vrednosti.

## ProcesConfig

Kroz `ProcesConfig` klijent može definisati svoje statuse i procesne korake.

Osnovni statusi ostaju:

```text
Zavedeno
U odobravanju
Odobreno
Odbijeno
Arhivirano
```

## Delovodne knjige

U App Config postoje delovodne knjige.

One sadrže aktivnu godinu i sledeći delovodni broj.

Kod zaključenja godine kreira se nova delovodna knjiga za sledeću godinu i menja se aktivna godina.
