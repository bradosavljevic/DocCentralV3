# SharePoint lista: Partneri

## Environment variable

```text
EV_DocCentralV3_lstPartneri
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstPartneri
```

## Namena

Lista `Partneri` sadrži poslovne partnere koji se biraju pri zavođenju dokumenata.

## Funkcionalnost

Korisnik koji zavodi dokumente može:

- kreirati partnera
- menjati partnera
- brisati partnera
- pregledati partnera

## Važno pravilo za brisanje

Brisanje partnera ne sme uticati na već zavedene dokumente.

Ako se partner obriše ili deaktivira, istorijski dokumenti u `Svi predmeti` moraju i dalje prikazivati podatke o partneru.

## Preporuka

U `Svi predmeti` čuvati snapshot vrednosti partnera, ne oslanjati se samo na lookup.
