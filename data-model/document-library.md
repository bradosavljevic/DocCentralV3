# SharePoint biblioteka: Dokumenti

## Environment variable

```text
EV_DocCentralV3_docDokumenti
```

Logical name:

```text
gpdoccen_EV_DocCentralV3_docDokumenti
```

## Pravilo skladištenja

Ne kreiraju se folderi.

Svi dokumenti i prilozi se kreiraju u root-u biblioteke.

## Glavni dokument i prilozi

Glavni dokument i prilozi se razlikuju metapodacima.

Prilog mora biti označen kao prilog i povezan sa glavnim dokumentom.

## Anti-overwrite pravilo

Zabranjeno je koristiti originalni naziv fajla kao jedini naziv fajla.

Sistemsko ime fajla mora biti unikatno.

Preporučeni format:

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

Za prilog:

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

## Preporučena metapolja

| Display name | Tip | Opis |
|---|---|---|
| DelovodniBroj | Single line of text | Delovodni broj |
| SviPredmetiId | Number | ID itema iz Svi predmeti |
| IsPrilog | Yes/No | Da li je fajl prilog |
| ParentDelovodniBroj | Single line of text | Delovodni broj glavnog dokumenta |
| ParentDocumentId | Number | ID glavnog dokumenta ili itema |
| OriginalFileName | Single line of text | Originalno ime fajla |
| SystemFileName | Single line of text | Sistemsko ime fajla |
| DocumentType | Single line of text / Choice | Tip dokumenta |
| DocumentStatus | Single line of text / Choice | Status dokumenta |
| CreatedByApp | Yes/No | Da li je kreirano iz aplikacije |
| CorrelationId | Single line of text | Veza sa flow procesom |

## Minimalno neophodna polja

```text
DelovodniBroj
SviPredmetiId
IsPrilog
OriginalFileName
SystemFileName
```
