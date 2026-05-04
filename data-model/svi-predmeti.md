# SharePoint lista: Svi predmeti

## Environment variable

```text
EV_DocCentralV3_lstSviPredmeti
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstSviPredmeti
```

## Namena

Lista `Svi predmeti` je glavna evidencija zavedenih dokumenata.

## Ključna pravila

- `DelovodniBroj` mora biti unikatan.
- Status dokumenta se vodi kroz polje `Stanje`.
- Dokumenti se ne pregledaju kroz Canvas aplikaciju kao opšta lista; gledaju se direktno u SharePoint-u.
- Canvas aplikacija prikazuje samo funkcionalne poglede, npr. dokumente za odobrenje, arhiviranje i unos.

## Osnovni statusi

```text
Zavedeno
U odobravanju
Odobreno
Odbijeno
Arhivirano
```

Kroz `ProcesConfig` klijent može definisati dodatne statuse.

## Preporučena dodatna snapshot polja za partnera

Da bi istorija ostala očuvana i posle brisanja partnera:

```text
PartnerId
PartnerNazivSnapshot
PartnerPIBSnapshot
PartnerMestoSnapshot
PartnerAdresaSnapshot
```

## Arhiviranje

Dokument može direktno iz `Zavedeno` u `Arhivirano`.

To je jedini direktni put arhiviranja.

## Odobravanje

Odobravanje menja `Stanje`.

Ako je odbijeno, status postaje `Odbijeno`.

## Permissions

Prava se fizički primenjuju na SharePoint item.

UI filtriranje nije dovoljno.
