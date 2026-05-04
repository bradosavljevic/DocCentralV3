# Security i permission model

## Osnovno pravilo

Korisnici imaju Read Only prava nad SharePoint-om.

Write operacije se izvršavaju kroz Power Automate pod service account-om.

## Service account

Service account ima Read/Write prava nad SharePoint listama i bibliotekama.

## Korisnici

Korisnici imaju Read pristup prema svojim pravima.

## Owners / Members / Viewers

- Owners: Read/Write
- Members: Read
- Viewers: Read
- Service account: Read/Write

## Admin grupa

Ne postoji posebna admin grupa kao obavezno pravilo.

Ako se uvede za konkretnog klijenta, mora biti konfiguraciona.

## Entra grupe

Mapa Entra grupa zavisi od klijenta.

Ne sme biti hardcoded.

Mora se čitati iz konfiguracije.

## Organizacione jedinice

Pristup dokumentima uvek je zasnovan na organizacionim jedinicama.

## SharePoint permissions

Prava se primenjuju fizički na:

- iteme u listama
- fajlove u bibliotekama

Koristi se item-level break inheritance.

## Canvas aplikacija

Aplikacija takođe primenjuje vidljivost i filtriranje, ali to nije jedini sigurnosni sloj.

Backend SharePoint prava moraju stvarno ograničiti pristup.
