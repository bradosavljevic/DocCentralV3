# 05 - AppConfig Model

`AppConfig` je centralno mesto za šifarnike i konfiguracije aplikacije.

## Core keys

- `DelovodneKnjige`
- `Stanja`
- `ProcesConfig`
- `TipoviDokumenta`
- `VrsteDokumenta`
- `OrganizacioneJedinice`
- `Translations`
- `Settings`
- `RegistracioneJedinice`
- `RokoviCuvanja`
- `ArhivskaListaKategorije`
- `KategorijeDokumentarnogMaterijala`
- `EfaktureParametri`
- `Users`
- `Valute`

## JSON row ID

Svaki zapis unutar JSON niza treba da ima `ID` kao GUID. Ovo omogućava Excel import, delta update, poređenje postojećih i novih zapisa i stabilan identitet reda.

## Schema validation

Svaki AppConfig update mora validirati očekivanu strukturu pre upisa. Ako schema nije validna, update se odbija, korisnik dobija jasnu poruku, a tehnički detalji idu u AuditLog.

## EfaktureParametri

Treba da postoji samo jedan unos: `EfaktureParametri`. Postojeći duplikat/varijanta `EFaktureParametri` je tehnički dug i ne sme se automatski brisati bez migracije.

## Excel import

Import šifarnika radi za pojedinačan šifarnik:
- ako zapis ne postoji, dodaj
- ako postoji, ažuriraj samo vrednosti koje su različite
- ne menjati nepromenjene zapise
- ne raditi blind replace celog JSON-a bez validacije
