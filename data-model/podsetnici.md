# SharePoint lista: Podsetnici

## Environment variable

```text
EV_DocCentralV3_lstPodsetnici
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstPodsetnici
```

## Namena

Lista čuva podsetnike koje korisnik kreira za sebe ili druge korisnike.

## Pravila

- Podsetnik šalje email jednom.
- Podsetnik se šalje na datum definisan u podsetniku.
- Podrazumevano vreme slanja je 08:00.
- Vreme slanja je sistemska konfiguracija, nije individualno po podsetniku.
- Korisnik može menjati i brisati podsetnike.
- Podsetnik može biti za jednog ili više korisnika.

## Flow

```text
CF_DocCentralV3_SendReminders
```
