# Template: Unikatno ime fajla

## Glavni dokument

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

## Prilog

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

## Power Automate izraz - primer

```text
concat(variables('SafeDelovodniBroj'), '_', guid(), '_', variables('SafeOriginalFileName'))
```

## Pravilo

Nikada ne koristiti samo originalni naziv fajla za Create file.
