# Known Issues & Technical Debt

## 1. Svrha dokumenta

Ovaj dokument evidentira poznate probleme, rizike i tehnički dug u postojećem DocCentral v6.0 rešenju.

Cilj dokumenta je da posluži kao osnova za:

- procenu rizika postojeće verzije
- planiranje enterprise unapređenja
- pripremu nove verzije aplikacije
- definisanje prioriteta u backlog-u
- usmeravanje Cloud Code / Claude Code razvoja

---

## 2. Status informacija

Informacije u ovom dokumentu su razdvojene po statusu:

- POTVRĐENO — direktno potvrđeno kroz dostavljene podatke ili korisničko objašnjenje
- PRETPOSTAVKA — zaključak izveden iz naziva lista, kolona, biblioteka ili procesa
- NEPOZNATO — podatak nije dostavljen ili nije moguće potvrditi

---

## 3. Najkritičniji poznati rizik

## 3.1 Dupli delovodni broj kod istovremenog zavođenja

Status: POTVRĐENO

Kritičnost: P0

Opis:

U sistemu postoji poslovni zahtev da više korisnika može istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovaj rizik je posebno naglašen jer se race condition problem već pojavio kada dvoje korisnika istovremeno zavode dokument.

Potvrđeno je da postoji poseban Power Automate flow zbog ovog problema:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Zaključak:

Dodela delovodnog broja je najvažniji enterprise rizik i mora biti centralni deo target arhitekture nove verzije.

Rizik:

Ako se delovodni broj generiše u Canvas aplikaciji, lokalnoj kolekciji ili prostim čitanjem poslednjeg broja iz SharePoint-a, može doći do duplikata.

Posledica:

- dva dokumenta mogu dobiti isti delovodni broj
- narušava se integritet pisarnice
- dokumentacija postaje pravno i poslovno nepouzdana
- audit trail može postati neupotrebljiv
- korisnici gube poverenje u sistem

Obavezna korekcija:

- centralizovati dodelu delovodnog broja
- koristiti server-side flow / service kao jedini izvor istine
- uvesti atomic / optimistic locking
- uvesti retry logiku
- uvesti unique constraint
- uvesti audit log za svaki pokušaj dodele broja
- vratiti kontrolisan odgovor Canvas aplikaciji

---

## 4. Canvas App rizici

## 4.1 Previše poslovne logike u Canvas aplikaciji

Status: PRETPOSTAVKA

Opis:

Na osnovu dosadašnje analize Power Platform obrazaca i postojanja posebnog flow-a za dodelu delovodnog broja, postoji rizik da deo poslovne logike živi u Canvas aplikaciji.

Canvas aplikacija treba da bude UI sloj, a ne glavni backend za kritične transakcije.

Rizik:

Ako Canvas App:

- računa sledeći delovodni broj
- oslanja se na lokalne kolekcije
- direktno upisuje kritične podatke
- radi finalnu validaciju bez backend potvrde

onda sistem nije dovoljno bezbedan za istovremeni rad više korisnika.

Preporuka:

Canvas App treba da:

- prikazuje formu
- radi osnovnu UI validaciju
- poziva Power Automate flow
- prikazuje rezultat korisniku
- prikazuje correlation ID kod greške

Canvas App ne treba da:

- samostalno dodeljuje delovodni broj
- garantuje jedinstvenost broja
- direktno obavlja kritične Create/Edit/Delete operacije
- bude jedini izvor poslovnih pravila

---

## 4.2 Nepotpuna poznata struktura Canvas aplikacije

Status: NEPOZNATO

Potvrđeno je:

- aplikacija je Canvas App
- početni ekran je `scrHome`
- postoji funkcionalnost zavođenja dokumenta
- pregled dokumenata se ne radi kroz poseban ekran, već direktno u SharePoint listi / biblioteci
- upload dokumenta ide preko flow-a `Upload Doc`

Nepoznato je:

- tačan broj ekrana
- svi nazivi ekrana
- nazivi glavnih kontrola
- OnStart logika
- App.Formulas / named formulas
- kolekcije koje se učitavaju
- globalne promenljive
- lokalne promenljive
- validaciona pravila
- detaljna navigacija
- error handling u aplikaciji
- response handling za flow-ove

Rizik:

Bez pune analize Canvas App paketa nije moguće dati kompletan tehnički opis UI logike.

Preporuka:

Za sledeću fazu analize potrebno je izvesti Canvas App iz solution-a i analizirati:

- `CanvasManifest.json`
- screen fajlove
- control tree
- formulas
- data sources
- flow references
- environment variables
- connection references

---

## 5. Power Automate rizici

## 5.1 Nedovoljno poznata flow arhitektura

Status: NEPOZNATO

Potvrđeni flow-ovi:

```text
Upload Doc
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Potvrđene funkcionalnosti:

- upload dokumenta ide preko Power Automate flow-a
- dodela delovodnog broja ide preko posebnog flow-a zbog race condition problema
- email dokumenti se obrađuju preko Power Automate flow-a koji čita shared mailbox i zavodi dokumente iz emailova sa attachmentom
- export šifarnika i konfiguracije iz AppConfig-a ide preko posebne funkcionalnosti

Nepoznato:

- tačni nazivi svih flow-ova
- trigger-i svih flow-ova
- input schema
- output schema
- konektori
- retry policy
- concurrency settings
- run after konfiguracija
- error handling
- logging
- da li se koristi service account
- da li flow-ovi vraćaju standardizovan response
- da li postoji centralizovan audit log

Rizik:

Bez standardizovane flow arhitekture može doći do:

- nedoslednih grešaka
- nepouzdanog response-a prema Canvas App
- teškoća u debugovanju
- timeout problema
- dupliranja logike
- nejasne odgovornosti između Canvas App i flow-ova

Preporuka:

Za svaki flow napraviti tehnički dokument sa:

- nazivom
- trigger-om
- ulazima
- izlazima
- konektorima
- SharePoint zavisnostima
- glavnim koracima
- concurrency podešavanjima
- retry podešavanjima
- error handling-om
- audit logging-om
- poznatim problemima

---

## 5.2 Race condition u procesu dodele broja

Status: POTVRĐENO

Opis:

Korisnik je eksplicitno naveo da postoji poseban flow zbog race condition problema kada dvoje korisnika istovremeno zavode dokument.

Flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Rizik:

Ako flow ne koristi dovoljno čvrst atomic / optimistic locking mehanizam, race condition se može ponoviti.

Posebno rizični obrasci:

- Get items -> Max number -> Create item
- čitanje poslednjeg broja bez unique constraint-a
- paralelno izvršavanje bez kontrole konkurentnosti
- retry bez provere stvarnog konflikta
- upis broja u više mesta bez transakcione kontrole

Preporuka:

U target arhitekturi proces dodele broja mora biti izolovan i tretiran kao poseban numbering service.

---

## 5.3 Mogući timeout kod dugih flow-ova

Status: PRETPOSTAVKA

Opis:

Power Automate flow-ovi koji se pozivaju iz Canvas aplikacije mogu praviti problem ako predugo traju pre nego što vrate odgovor aplikaciji.

Rizik:

- Canvas App može dobiti timeout
- korisnik ne zna da li je dokument uspešno zaveden
- može doći do ponovnog slanja istog zahteva
- ponovni zahtev može izazvati duple pokušaje upisa

Preporuka:

Kritični flow-ovi treba da:

- brzo vrate kontrolisan odgovor kada je moguće
- imaju correlation ID
- upisuju status u audit log
- spreče dupli submit
- imaju idempotency key za kritične operacije

---

## 6. SharePoint data model rizici

## 6.1 Nedovoljno indeksirana polja

Status: PRETPOSTAVKA

Opis:

U dostavljenim XML metadata odgovorima za više relevantnih polja vidi se da `Indexed` nije uključen.

Primeri važnih polja koja treba razmotriti za indeksiranje:

- `DelovodniBroj`
- `EdokumentID`
- `Edokument`
- `Attachment`
- `RezervisaniBroj`
- `DatumRezervacije`
- `Title` u pojedinim listama
- statusna i lookup polja koja se koriste u filterima

Rizik:

Ako liste i biblioteke rastu, neindeksirana polja mogu izazvati:

- spore upite
- probleme sa delegacijom
- SharePoint list view threshold probleme
- sporije flow izvršavanje
- sporiji prikaz u SharePoint listama

Preporuka:

Napraviti poseban indexing plan za sve liste i biblioteke.

---

## 6.2 Unique constraint nije potvrđen za kritična polja

Status: NEPOZNATO

Opis:

Za kritični zahtev jedinstvenog delovodnog broja mora postojati jedinstvena kontrola.

Nepoznato je da li je unique constraint uključen na:

- finalnom polju `DelovodniBroj`
- polju `RezervisaniBroj`
- kombinaciji godina + broj
- kombinaciji tip dokumenta + godina + broj

Rizik:

Bez unique constraint-a, aplikaciona logika može pogrešiti i dozvoliti duplikat.

Preporuka:

U novoj verziji obavezno definisati jedinstvenost na nivou data modela.

Minimalno:

- finalni `DelovodniBroj` mora biti unique
- rezervisani broj mora biti unique u okviru relevantnog scope-a
- scope mora biti jasno definisan: godina, tip dokumenta, organizaciona jedinica ili globalno

---

## 6.3 SharePoint kao transakcioni sistem

Status: PRETPOSTAVKA

Opis:

SharePoint Online se koristi kao primarni data layer i storage layer.

Rizik:

SharePoint nije klasična relaciona transakciona baza.

Kod enterprise pisarnice treba posebno voditi računa o:

- konkurentnom upisu
- list view threshold-u
- throttling-u
- audit-u
- jedinstvenosti brojeva
- permissions modelu
- delegaciji
- performance-u
- retry logici

Preporuka:

Zadržati SharePoint ako je to poslovno i licencno poželjno, ali kritične procese dizajnirati sa jasnim ograničenjima SharePoint-a.

---

## 7. AppConfig rizici

## 7.1 AppConfig kao centralna konfiguracija

Status: POTVRĐENO DELIMIČNO

Potvrđeno:

Lista `AppConfig` postoji i sadrži polja:

- `Title`
- `Config`
- `ColumnHeader`

Korisnik je potvrdio da se svi šifarnici i config JSON nalaze u `AppConfig` listi.

Nepoznato:

- tačan JSON format
- broj konfiguracionih zapisa
- struktura šifarnika
- da li postoji verzionisanje konfiguracije
- da li postoji validacija JSON-a
- da li postoji export/import mehanizam sa kontrolom verzije
- da li postoji rollback konfiguracije

Rizik:

Ako AppConfig nema strogu strukturu i validaciju:

- pogrešan JSON može srušiti deo aplikacije
- promene konfiguracije mogu uticati na produkciju bez kontrole
- nema jasnog rollback-a
- nema audit-a nad promenama šifarnika

Preporuka:

Uvesti:

- JSON schema po tipu konfiguracije
- version polje
- active/inactive status
- changed by / changed at
- config export
- config validation flow
- backup pre izmene
- audit log za izmene

---

## 8. Email intake rizici

## 8.1 Obrada email dokumenata iz shared mailbox-a

Status: POTVRĐENO DELIMIČNO

Korisnik je potvrdio:

Email dokumenti su posebna funkcionalnost gde Power Automate čita shared mailbox i zavodi dokumente iz emailova sa attachmentom.

Biblioteka:

```text
EmailDocuments
```

Poznata polja:

- `FileLeafRef`
- `Title`
- `_ExtendedDescription`
- `PosiljalacEmail`
- `Posiljalac`
- `MediaServiceImageTags`
- `ContentType`

Nepoznato:

- naziv flow-a
- shared mailbox adresa
- trigger
- da li se obrađuju samo emailovi sa attachmentom
- kako se sprečava dupli import istog emaila
- da li se koristi MessageId
- kako se čuva status obrade
- kako se obrađuju greške
- gde se loguju greške
- da li se čuva originalni email metadata

Rizik:

Bez idempotency kontrole, isti email ili attachment može biti obrađen više puta.

Preporuka:

Za email intake uvesti:

- `InternetMessageId` ili Graph message ID
- hash attachmenta po potrebi
- status obrade
- processed timestamp
- error log
- retry policy
- dead-letter logiku za neuspešne emailove

---

## 9. Export rizici

## 9.1 Export šifarnika i konfiguracije

Status: POTVRĐENO DELIMIČNO

Korisnik je potvrdio:

Export se odnosi na šifarnike i konfiguracije iz `AppConfig`.

Biblioteka:

```text
Exports
```

Nepoznato:

- format exporta
- naziv flow-a
- da li export ide u Excel, JSON, CSV ili drugi format
- da li se export verzioniše
- da li se export koristi za backup
- da li postoji import nazad u sistem
- da li postoji kontrola pristupa export fajlovima

Rizik:

Ako export sadrži konfiguraciju sistema, treba kontrolisati ko može da ga generiše i preuzima.

Preporuka:

Uvesti:

- naziv export fajla sa datumom i verzijom
- audit za svaki export
- kontrolu pristupa
- čuvanje source config verzije
- optional checksum / hash fajla

---

## 10. Security i permissions rizici

## 10.1 Direktan pristup SharePoint-u

Status: PRETPOSTAVKA

Opis:

Pregled dokumenata se radi direktno kroz SharePoint listu / biblioteku.

Rizik:

Ako korisnici imaju šira SharePoint prava nego što je potrebno, mogu direktno menjati podatke mimo Canvas App i Power Automate kontrole.

Preporuka:

Ciljni model:

- korisnici imaju Read Only nad kritičnim listama i bibliotekama
- Create/Edit/Delete ide preko Power Automate flow-ova
- flow-ovi rade kroz service account
- kritične liste imaju ograničene dozvole
- administratorske funkcije su vezane za Entra ID grupe

---

## 10.2 Audit nije potvrđen

Status: NEPOZNATO

Nepoznato je da li postoji centralizovan audit log za:

- zavođenje dokumenta
- dodelu delovodnog broja
- upload dokumenta
- email intake
- export konfiguracije
- greške flow-ova
- izmene AppConfig-a

Rizik:

Bez audit loga je teško dokazati:

- ko je šta uradio
- kada je broj dodeljen
- zašto je došlo do greške
- da li je retry uspeo
- da li je dokument uploadovan
- koji flow run je obradio zahtev

Preporuka:

Uvesti centralnu listu:

```text
AuditLog
```

Minimalna polja:

- Title
- CorrelationId
- FlowName
- Action
- Status
- UserEmail
- StartedAt
- FinishedAt
- DurationMs
- DelovodniBroj
- DocumentItemId
- DocumentUrl
- ErrorCode
- ErrorMessage
- RetryCount
- RawPayload

---

## 11. ALM i deployment rizici

## 11.1 Nepoznat ALM model

Status: NEPOZNATO

Nepoznato:

- da li postoji Dev/Test/Prod environment
- da li se koristi managed solution
- da li se koristi connection references
- da li se koriste environment variables
- da li postoji deployment pipeline
- da li postoji GitHub repo za source dokumentaciju i artefakte
- da li se solution exportuje redovno

Rizik:

Bez ALM procesa:

- promene mogu biti ručne i teško ponovljive
- deployment može zavisiti od jednog korisnika
- konekcije mogu pucati pri importu
- teško je vratiti prethodnu verziju

Preporuka:

Uvesti ALM minimum:

- Dev environment
- Test/UAT environment
- Prod environment
- solution versioning
- connection references
- environment variables
- deployment checklist
- export solution backup
- GitHub dokumentacija
- release notes

---

## 12. Prioriteti za rešavanje

## P0 — Kritično

- concurrency-safe dodela delovodnog broja
- unique constraint za finalni delovodni broj
- retry i controlled error response
- audit log za numbering flow
- sprečavanje duplog submit-a iz Canvas App

## P1 — Visok prioritet

- standardizacija Power Automate response formata
- audit log za upload, email intake i export
- security model sa Read Only SharePoint pristupom za korisnike
- AppConfig schema i validacija
- indeksiranje ključnih SharePoint kolona

## P2 — Srednji prioritet

- kompletna dokumentacija svih flow-ova
- kompletna analiza Canvas App ekrana i formula
- export/import konfiguracije
- environment variables
- deployment checklist

## P3 — Niži prioritet

- UI polish
- dodatni dashboard
- napredna reporting funkcionalnost
- dodatna lokalizacija
- prošireni admin panel

---

## 13. Otvorene stavke za potvrdu

Potrebno je potvrditi:

1. Da li `DelovodniBroj` u glavnoj listi / biblioteci ima unique constraint?
2. Da li `RezervisaniBroj` ima unique constraint?
3. Koji je tačan scope delovodnog broja: globalno, po godini, po tipu dokumenta ili po organizacionoj jedinici?
4. Koji flow finalno upisuje dokument u glavnu evidenciju?
5. Koji flow finalno upisuje dokument u `Shared Documents`?
6. Koji flow obrađuje shared mailbox?
7. Koji flow radi export konfiguracije?
8. Da li postoji centralni audit log?
9. Da li postoji service account?
10. Koja prava imaju obični korisnici nad SharePoint listama i bibliotekama?
11. Da li se AppConfig menja ručno ili kroz administrativni ekran?
12. Da li postoji Dev/Test/Prod okruženje?
13. Da li se koristi managed solution?
14. Da li postoje environment variables?
15. Da li postoje connection references u solution-u?

---

## 14. Zaključak

Najveći poznati tehnički rizik u postojećem DocCentral v6.0 rešenju je dodela jedinstvenog delovodnog broja u situaciji kada više korisnika istovremeno zavodi dokumente.

To mora biti rešeno kao enterprise backend proces, ne kao UI logika.

Nova verzija mora imati:

- centralizovan numbering service
- concurrency-safe rezervaciju broja
- unique constraint
- retry logiku
- audit log
- standardizovan response
- jasan security model
- bolju dokumentovanost flow-ova
- validiran AppConfig model
- ALM proces
