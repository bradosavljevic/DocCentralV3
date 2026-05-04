# SharePoint lista: Rezervisani brojevi

## Environment variable

```text
EV_DocCentralV3_lstRezervisaniBrojevi
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi
```

## Namena

Lista čuva rezervisane delovodne brojeve.

## Pravila

- Rezervisani broj kreira korisnik koji zavodi dokumente.
- Rezervisani broj može menjati korisnik koji zavodi dokumente.
- Nakon rezervacije, rezervisani broj se ne može ručno obrisati.
- Rezervisani broj se briše automatski tek kada se iskoristi.
- Rezervisani broj važi samo za jednu godinu.
- Rezervisani broj može se iskoristiti za bilo koji tip dokumenta.
- Svi korisnici koji zavode dokumente vide sve rezervisane brojeve.

## Korišćenje rezervisanog broja

Kada korisnik izabere rezervisani broj:

1. bira datum zavođenja
2. sistem proverava validnost broja
3. sistem kreira dokument
4. sistem briše rezervisani broj nakon uspešnog zavođenja
5. sistem loguje korišćenje rezervisanog broja

## Flow

```text
CF_DocCentralV3_UseReservedNumber
```
