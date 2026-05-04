# Proces: Odobravanje dokumenta

## Osnovna pravila

Dokument ide na odobrenje jednom korisniku u trenutku.

Ako postoji više korisnika, odobravanje je sekvencijalno.

Ako se odobrenje šalje na grupu, više korisnika dobija informaciju, ali prvi korisnik koji odobri završava taj korak.

## Flow-ovi

| Flow | Namena |
|---|---|
| CF_DocCentralV3_SendForApproval | Pokreće approval |
| CF_DocCentralV3_ProcessApprovalResponse | Obrađuje odgovor |
| CF_DocCentralV3_LogEvent | Loguje događaje |

## Statusi

| Događaj | Status |
|---|---|
| Dokument poslat na odobrenje | U odobravanju |
| Dokument odobren | Odobreno ili sledeći status iz procesa |
| Dokument odbijen | Odbijeno |
| Dokument vraćen inicijatoru | Odbijeno |

## Odbijanje

Ako korisnik odbije dokument:

1. Dokument dobija status `Odbijeno`.
2. Dokument se vraća inicijatoru.
3. Inicijator može izmeniti dokument ili metapodatke.
4. Inicijator može ponovo pokrenuti proces odobrenja.

## Vidljivost

U Canvas aplikaciji korisnik vidi dokumente koje još nije odobrio.

U SharePoint listi korisnik vidi dokumente koji su za njega ili gde je on ili njegova grupa učestvovala u procesu odobrenja.

## Polje statusa

Status dokumenta se menja kroz polje:

```text
Stanje
```
