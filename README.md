# DocCentralV3 dokumentacija

## Status dokumentacije

Ova dokumentacija predstavlja radnu osnovu za razvoj nove verzije rešenja **DocCentralV3** kroz Claude Code.

Cilj dokumentacije nije samo opis postojećeg rešenja, već priprema jasnog funkcionalnog, arhitektonskog i tehničkog brief-a za izradu nove verzije aplikacije.

## Obuhvat rešenja

DocCentralV3 je white-label Power Platform rešenje za elektronsku pisarnicu / zavođenje dokumenata.

Rešenje se prilagođava po klijentu kroz konfiguraciju, šifarnike, SharePoint liste, biblioteke, Entra grupe i App Config.

## Ciljana arhitektura

- Power Apps Canvas aplikacija
- Power Automate cloud flow-ovi
- SharePoint Online liste i biblioteke
- App Config lista za šifarnike i konfiguraciju
- SharePoint liste za audit/logging
- SharePoint liste za dokumente, partnere, podsetnike i rezervisane brojeve
- Entra ID grupe za prava pristupa
- Service account za izvršavanje write operacija
- Korisnici imaju Read Only pristup nad SharePoint podacima
- Upisi, izmene i brisanja idu preko Power Automate flow-ova

## Važno pravilo za Claude Code

Sav kod koji Claude Code generiše mora biti smešten u folder:

```text
PACode
```

Dokumentacija, šabloni, checkliste i skills fajlovi ostaju u svojim folderima.

## Naziv solution-a

```text
DocCentralV3
```

## Publisher

```text
GoProDocCentral
```

## Version

```text
3.0.0.0
```

## Canvas aplikacija

Display name:

```text
DocCentralV3
```

Logical name iz trenutnog okruženja:

```text
gpdoccen_doccentralv3_d98ba
```

Claude Code sme da odluči finalni broj ekrana, nazive ekrana i finalni UX dizajn na osnovu zahteva.

## Environment variables

Kreirane environment variables:

| Display name | Logical name | Namena |
|---|---|---|
| EV_DocCentralV3_SharePointSite | gpdoccen_EV_DocCentralV3_SharePointSite | URL SharePoint sajta |
| EV_DocCentralV3_lstSviPredmeti | gpdoccen_EV_DocCentralV3_lstSviPredmeti | Lista Svi predmeti |
| EV_DocCentralV3_lstPartneri | gpdoccen_EV_DocCentralV3_lstPartneri | Lista Partneri |
| EV_DocCentralV3_lstAppConfig | gpdoccen_EV_DocCentralV3_lstAppConfig | Lista App Config |
| EV_DocCentralV3_lstPodsetnici | gpdoccen_EV_DocCentralV3_lstPodsetnici | Lista Podsetnici |
| EV_DocCentralV3_lstRezervisaniBrojevi | gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi | Lista Rezervisani brojevi |
| EV_DocCentralV3_lstAuditLog | gpdoccen_EV_DocCentralV3_lstAuditLog | Audit log lista |
| EV_DocCentralV3_docDokumenti | gpdoccen_EV_DocCentralV3_docDokumenti | Glavna biblioteka dokumenata |
| EV_DocCentralV3_docEmailDocs | gpdoccen_EV_DocCentralV3_docEmailDocs | Biblioteka/email dokumenti |
| EV_DocCentralV3_docExports | gpdoccen_EV_DocCentralV3_docExports | Biblioteka za exporte |

## Connection references

Kreirane connection references:

| Display name | Logical name | Konektor |
|---|---|---|
| CR_DocCentralV3_SharePoint | gpdoccen_CR_DocCentralV3_SharePoint | SharePoint |
| CR_DocCentralV3_Outlook | gpdoccen_CR_DocCentralV3_Outlook | Outlook |
| CR_DocCentralV3_Office365Users | gpdoccen_CR_DocCentralV3_Office365Users | Office 365 Users |
| CR_DocCentralV3_Office365Groups | gpdoccen_CR_DocCentralV3_Office365Groups | Office 365 Groups |
| CR_DocCentralV3_OneDrive | gpdoccen_CR_DocCentralV3_OneDrive | OneDrive |
| CR_DocCentralV3_Excel | gpdoccen_CR_DocCentralV3_Excel | Excel |

## Cloud flow-ovi

Kreirani/predviđeni cloud flow-ovi:

| Flow | Namena |
|---|---|
| CF_DocCentralV3_CreateDocument | Centralni flow za zavođenje dokumenta |
| CF_DocCentralV3_GenerateRegistryNumber | Generisanje sledećeg delovodnog broja |
| CF_DocCentralV3_UseReservedNumber | Korišćenje rezervisanog delovodnog broja |
| CF_DocCentralV3_AssignPermissions | Dodela prava na iteme/fajlove |
| CF_DocCentralV3_SendForApproval | Slanje dokumenta na odobrenje |
| CF_DocCentralV3_ProcessApprovalResponse | Obrada odgovora odobravača |
| CF_DocCentralV3_SendReminders | Slanje email podsetnika |
| CF_DocCentralV3_ArchiveDocument | Arhiviranje dokumenta |
| CF_DocCentralV3_CloseRegistryYear | Zaključenje delovodne godine |
| CF_DocCentralV3_ExportAppConfig | Export šifarnika i konfiguracija |
| CF_DocCentralV3_GenerateArchiveBookPdf | Generisanje PDF arhivske knjige |
| CF_DocCentralV3_LogEvent | Upis audit log događaja |

Flow `CF_DocCentralV3_CreateDocumentFolder` se ne koristi, jer dokumenti ne idu u foldere.

## Pravilo za dokument biblioteku

Dokumenti se ne smeštaju u posebne foldere.

Svi dokumenti i prilozi se kreiraju u root-u SharePoint biblioteke.

Ako dokument ima prilog, prilog se označava metapodatkom da je prilog i povezuje se sa glavnim dokumentom preko metapodataka.

## Pravilo protiv overwrite-a

Flow nikada ne sme kreirati fajl sa originalnim korisničkim imenom kao jedinim nazivom fajla.

Svaki fajl mora imati sistemski generisano unikatno ime.

Preporučeni format:

```text
{SafeDelovodniBroj}_{guid()}_{SafeOriginalFileName}
```

Za prilog:

```text
{SafeDelovodniBroj}_PRILOG_{guid()}_{SafeOriginalFileName}
```

Originalno ime fajla se čuva u posebnom metapodatku:

```text
OriginalFileName
```

Sistemsko ime se čuva u:

```text
SystemFileName
```

## Glavni poslovni procesi

- Unos novog dokumenta
- Korišćenje rezervisanog broja
- Generisanje sledećeg broja
- Odobravanje dokumenta
- Odbijanje dokumenta
- Arhiviranje dokumenta
- Zaključenje godine
- Upravljanje partnerima
- Upravljanje šifarnicima
- Podsetnici
- Export šifarnika
- Generisanje PDF arhivske knjige
- Audit logging

## Statusi dokumenata

Osnovni statusi su:

```text
Zavedeno
U odobravanju
Odobreno
Odbijeno
Arhivirano
```

Kroz `ProcesConfig` klijent može definisati dodatne ili prilagođene statuse, ali osnovni statusi ostaju referentni minimum.

## Navigacija aplikacije

U aplikaciji ne postoji dashboard.

U aplikaciji ne postoji ekran za pregled svih dokumenata.

Dokumenti se gledaju direktno u SharePoint listama/bibliotekama.

Aplikacija mora imati funkcionalne oblasti:

- Novi predmet / unos dokumenta
- Dokumenti koje korisnik treba da odobri
- Podsetnici
- Arhiviranje
- Zaključenje godine
- Administracija
- Partneri kao zasebna stavka menija, ne deo administracije

## Arhiviranje

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen i predstavlja jedini direktni put arhiviranja.

Ekran za arhiviranje prikazuje dokumente u statusu `Zavedeno` za tekuću godinu i omogućava arhiviranje sa odgovarajućim arhivskim znacima.

## Zaključenje godine

Zaključana godina se nikada ne može ponovo otključati.

Uslovi za zaključenje godine:

- svi dokumenti za godinu moraju biti u statusu `Arhivirano`
- ne sme postojati dokument u bilo kom drugom statusu
- lista rezervisanih brojeva mora biti prazna
- kreira se nova delovodna knjiga za sledeću godinu
- aktivna godina se menja u App Config
- potvrda administratora nije potrebna

## Odobravanje

Dokument ide na odobrenje jednom korisniku u trenutku.

Ako ima više korisnika, proces je sekvencijalan.

Moguće je poslati approval na grupu; više korisnika dobija informaciju, ali prvi korisnik koji odobri završava taj korak.

Odobrenje menja status dokumenta kroz polje `Stanje`.

Ako je odobrenje odbijeno, dokument dobija status `Odbijeno`, vraća se inicijatoru, a inicijator nakon korekcije dokumenta ili metapodataka može ponovo pokrenuti proces odobrenja.

## Partneri

Partneri su zasebna funkcionalnost u meniju.

Korisnik koji zavodi dokumente može da:

- kreira partnera
- menja partnera
- briše partnera
- pregleda partnera

Ako se partner briše, brisanje ne sme narušiti istorijske dokumente u `Svi predmeti`.

Partner više ne sme biti dostupan za novo biranje, ali ranije zavedeni dokumenti moraju zadržati istorijske podatke o partneru.

## Podsetnici

Korisnik može kreirati podsetnik za sebe, jednog korisnika ili više korisnika.

Podsetnik šalje email jednom, na dan definisanog podsetnika.

Korisnik ne definiše individualno vreme slanja; sistemsko vreme slanja je zajedničko, podrazumevano 08:00.

Korisnik može menjati i brisati podsetnike.

Ne postoje statusi podsetnika tipa `Aktivan`, `Poslat`, `Otkazan`, osim ako se kasnije uvedu kao tehnička polja.

## Audit log

Audit se implementira kroz SharePoint listu `AuditLog`.

Obavezno logovati:

- neuspešno kreiranje dokumenta
- neuspešan pokušaj generisanja delovodnog broja
- uspešno generisan delovodni broj
- neuspešno generisanje broja
- korišćenje rezervisanog broja
- arhiviranje
- zaključenje godine
- grešku Power Automate flow-a

## PDF generisanje

PDF generisanje ulazi u prvu verziju samo za Arhivsku knjigu.

Ne generišu se svi dokumenti kao PDF u prvoj verziji.

## Deployment

Deployment model je ručni import solution-a po tenantima.

Nema migracije postojećih produkcionih podataka za nove klijente, jer novi klijenti kreću od praznog okruženja.

Postojeći klijenti već koriste postojeće liste.

## Standardi kvaliteta

Claude Code mora poštovati:

- responsive design
- čist App Checker
- formula-level error management
- `IfError` oko Patch/Flow poziva
- `gbl`, `loc`, `col` naming konvencije
- delegabilne formule
- bez live search query-ja na svaki karakter
- bez referenciranja kontrola sa drugih ekrana
- bez nested galleries osim ako je opravdano

## Trenutne poznate odluke

| Tema | Odluka |
|---|---|
| Finalni UX dizajn | Claude Code odlučuje prema best practice |
| Entra grupa po klijentu | Mora biti konfiguraciono, ne hardcoded |
| App Config | Produkcioni šifarnici dati kroz AppConfig.csv |
| Audit | SharePoint lista |
| PDF | Prva verzija samo Arhivska knjiga |
| Migracija | Nije potrebna za nove klijente |
| Deployment | Ručni solution import |
| Test matrica | Mora se napraviti |
| Kod | Sav generisani kod ide u `PACode` |
