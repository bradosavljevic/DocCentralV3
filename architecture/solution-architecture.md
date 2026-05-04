# DocCentralV3 - arhitektura rešenja

## Arhitektonski princip

DocCentralV3 koristi Power Apps kao korisnički interfejs, Power Automate kao servisni sloj i SharePoint Online kao data/document storage.

Korisnik ne treba direktno da upisuje u SharePoint liste i biblioteke.

## Komponente

| Komponenta | Uloga |
|---|---|
| Canvas App `DocCentralV3` | Korisnički interfejs |
| SharePoint liste | Poslovni podaci, konfiguracija, šifarnici, log |
| SharePoint biblioteke | Dokumenti, prilozi, exporti, email dokumenti |
| Power Automate | Svi write procesi i backend logika |
| Service account | Izvršava write operacije |
| Entra ID grupe | Kontrola pristupa |
| App Config | Konfiguracija, šifarnici, procesi i delovodne knjige |

## Power Apps pravila

- Aplikacija mora biti responzivna.
- Finalni broj ekrana određuje Claude Code na osnovu funkcionalnosti.
- U aplikaciji nema dashboard-a.
- U aplikaciji nema pregleda svih dokumenata.
- Dokumenti se gledaju direktno u SharePoint listama/bibliotekama.
- Aplikacija mora prikazati dokumente koje korisnik treba da odobri.
- Aplikacija mora prikazati podsetnike.
- Aplikacija mora imati unos novog dokumenta.
- Aplikacija mora imati arhiviranje.
- Aplikacija mora imati administraciju šifarnika.
- Partneri su zasebna stavka menija.

## Power Automate pravila

Power Automate flow-ovi su backend sloj.

Svi flow-ovi moraju koristiti postojeće connection references:

- `CR_DocCentralV3_SharePoint`
- `CR_DocCentralV3_Outlook`
- `CR_DocCentralV3_Office365Users`
- `CR_DocCentralV3_Office365Groups`
- `CR_DocCentralV3_OneDrive`
- `CR_DocCentralV3_Excel`

## SharePoint pravila

- Korisnici imaju Read Only pristup.
- Service account ima Read/Write.
- Item/file permissions se fizički primenjuju na backend-u.
- UI filtriranje nije dovoljno za security.
- Prava se primenjuju i u Canvas aplikaciji i u SharePoint-u.
- Dokumenti se čuvaju u root-u biblioteke, bez foldera.

## Dokumenti i prilozi

Ne kreiraju se folderi za dokumente.

Svi fajlovi se kreiraju u root-u biblioteke `EV_DocCentralV3_docDokumenti`.

Glavni dokument i prilozi razlikuju se po metapodacima.

Prilog mora imati oznaku da je prilog i mora biti povezan sa glavnim dokumentom.

## Anti-overwrite pravilo

Svaki fajl mora imati unikatno sistemsko ime.

Preporuka:

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

Prilog:

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

## Audit

Audit se čuva u SharePoint listi preko `EV_DocCentralV3_lstAuditLog`.

Flow `CF_DocCentralV3_LogEvent` je centralni flow za logovanje.
