# Proces: Unos novog dokumenta

## Opis

Ovo je centralni poslovni proces aplikacije.

Unos novog dokumenta se izvršava kroz Canvas aplikaciju, ali kompletna backend logika mora biti u flow-u:

```text
CF_DocCentralV3_CreateDocument
```

## Koraci procesa

1. Korisnik otvara aplikaciju.
2. Korisnik bira tip dokumenta.
3. Korisnik popunjava obavezna polja.
4. Korisnik dodaje prilog.
5. Korisnik bira da li koristi rezervisani broj ili generiše sledeći broj u nizu.
6. Ako koristi rezervisani broj, bira datum zavođenja.
7. Ako koristi sledeći broj u nizu, datum zavođenja se automatski postavlja na današnji datum.
8. Korisnik klikće `Zavedi`.
9. Sistem proverava validaciju.
10. Sistem preuzima ili generiše delovodni broj.
11. Sistem kreira SharePoint item u listi `Svi predmeti`.
12. Sistem kreira dokument u root-u SharePoint biblioteke.
13. Ako postoje prilozi, sistem kreira priloge u root-u SharePoint biblioteke.
14. Sistem dodeljuje prava na item i fajlove.
15. Ako broj nije rezervisan, sistem povećava brojač za sledeće zavođenje.
16. Ako je broj rezervisan, sistem nakon uspešnog zavođenja uklanja iskorišćeni broj iz liste rezervisanih brojeva.
17. Sistem upisuje audit log.
18. Sistem vraća poruku korisniku.

## Važna odluka

Ne kreira se folder za dokument.

Svi dokumenti i prilozi se kreiraju u root-u biblioteke.

## Pravilo za naziv fajla

Nikada ne koristiti originalni naziv fajla kao jedini naziv fajla u SharePoint biblioteci.

Obavezno kreirati sistemski unikatan naziv.

Preporučeni format za glavni dokument:

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

Preporučeni format za prilog:

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

## Minimalni metapodaci na fajlu

| Kolona | Namena |
|---|---|
| DelovodniBroj | Veza sa predmetom |
| SviPredmetiId | ID itema iz liste Svi predmeti |
| IsPrilog | Da li je fajl prilog |
| ParentDelovodniBroj | Delovodni broj glavnog dokumenta |
| ParentDocumentId | ID glavnog dokumenta ili itema |
| OriginalFileName | Originalno ime fajla |
| SystemFileName | Sistemsko ime fajla |
| DocumentType | Tip dokumenta |
| DocumentStatus | Status dokumenta |
| CreatedByApp | Oznaka da je fajl kreiran iz aplikacije |

## Tok za rezervisani broj

Ako korisnik koristi rezervisani broj:

1. Flow proverava da broj postoji.
2. Flow proverava da pripada aktivnoj godini.
3. Flow proverava da broj nije već iskorišćen.
4. Flow kreira dokument.
5. Tek nakon uspešnog kreiranja dokumenta briše rezervisani broj iz liste.
6. Flow loguje korišćenje rezervisanog broja.

## Tok za novi broj

Ako korisnik generiše sledeći broj:

1. Flow čita aktivnu delovodnu knjigu iz App Config.
2. Flow generiše broj na concurrency-safe način.
3. Flow proverava unique constraint na `DelovodniBroj`.
4. Flow kreira dokument.
5. Flow povećava brojač za sledeće zavođenje.
6. Flow loguje rezultat.

## Error handling

Ako bilo koji korak ne uspe:

- dokument ne sme ostati delimično zaveden bez jasnog audit zapisa
- korisnik mora dobiti jasnu poruku
- greška mora biti upisana u `AuditLog`
- flow mora vratiti strukturisan response Canvas aplikaciji
