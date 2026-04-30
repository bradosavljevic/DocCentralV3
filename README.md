# DocCentral v6.0 — Tehnička dokumentacija i razvojna osnova

## 1. Svrha dokumentacije

Ovaj repozitorijum sadrži tehničku dokumentaciju za Power Platform rešenje **DocCentral v6.0**.

Cilj dokumentacije je:

1. Razumevanje postojeće aplikacije.

2. Dokumentovanje Power Apps, Power Automate i SharePoint data modela.

3. Identifikovanje tehničkog duga i rizika.

4. Priprema osnove za razvoj nove enterprise verzije aplikacije.

5. Priprema razvojnih instrukcija za Cloud Code / Claude Code / AI-assisted development.

---

## 2. Opis rešenja

DocCentral v6.0 je Power Platform rešenje za elektronsku pisarnicu / centralnu evidenciju dokumenata.

Na osnovu dostavljenih SharePoint REST XML metadata odgovora, rešenje koristi SharePoint Online kao primarni data layer i storage layer.

Identifikovane komponente:

- SharePoint liste

- SharePoint document libraries

- Power Apps Canvas aplikacija

- Power Automate flow-ovi

- konfiguraciona lista `AppConfig`

- lista za rezervaciju brojeva `RezervisaniBrojevi`

- biblioteke dokumenata

- email dokument biblioteka

- export biblioteka

---

## 3. Identifikovane SharePoint lokacije

Site:

```text

https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0

---
## REST API base:
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/

4. Identifikovane SharePoint liste i biblioteke

Na osnovu dostavljenih XML metadata odgovora identifikovane su sledeće liste/biblioteke:
Naziv

Tip

Scope

Namena

RezervisaniBrojevi

SharePoint lista

/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi

Rezervacija / kontrola delovodnih brojeva

AppConfig

SharePoint lista

/sites/DocumentCentralv6.0/Lists/AppConfig

Centralna konfiguracija aplikacije

Shared Documents

Document library

/sites/DocumentCentralv6.0/Shared Documents

Glavna biblioteka dokumenata

EmailDocuments

Document library

/sites/DocumentCentralv6.0/EmailDocuments

Dokumenti nastali iz email procesa

Exports

Document library

/sites/DocumentCentralv6.0/Exports

Exportovani fajlovi / izveštaji


5. Ključni poslovni zahtev

Najvažniji enterprise zahtev:

Više korisnika mora moći istovremeno da zavode dokumenta,aPower Apps Canvas App

        |

        | poziva / čita / šalje podatke

        v

Power Automate flows

        |

        | kreiraju, ažuriraju, validiraju

        v

SharePoint Online

        |

        | liste + biblioteke

        v

Dokumenti, konfiguracija, delovodni brojevi, exportili sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.


Ovaj zahtev ima najviši prioritet u novoj verziji rešenja.

To znači:

* delovodni broj se ne sme generisati samo u Canvas aplikaciji
* delovodni broj se ne sme generisati lokalno u kolekciji
* delovodni broj se ne sme zasnivati samo na poslednjem pročitanom broju iz SharePoint liste bez kontrole konkurentnosti
* mora postojati centralizovan servis za generisanje brojeva
* mora postojati atomic / optimistic locking mehanizam
* mora postojati retry logika
* mora postojati audit log
* mora postojati unique constraint na finalnom delovodnom broju

6. Visok nivo arhitekture

Trenutna arhitektura se može opisati ovako:

Power Apps Canvas App

        |

        | poziva / čita / šalje podatke

        v

Power Automate flows

        |

        | kreiraju, ažuriraju, validiraju

        v

SharePoint Online

        |

        | liste + biblioteke

        v

Dokumenti, konfiguracija, delovodni brojevi, exporti

7. Glavni SharePoint elementi

7.1 AppConfig

Lista AppConfig sadrži centralnu konfiguraciju aplikacije.

Identifikovana polja:
Internal name

Display name

Tip

Title

Title

Single line of text

Config

Config

Multiple lines of text

ColumnHeader

ColumnHeader

Multiple lines of text

ID

ID

Counter

Created

Created

Date and Time

Modified

Modified

Date and Time

Author

Created By

Person or Group

Editor

Modified By

Person or Group

Pretpostavka:

Config polje sadrži JSON konfiguracije koje aplikacija koristi za inicijalizaciju kolekcija, podešavanja, prevode i poslovna pravila.

Status: POTVRĐENO delimično
Razlog: XML potvrđuje postojanje kolone Config, ali konkretan sadržaj JSON konfiguracije nije dostavljen.

⸻

7.2 RezervisaniBrojevi

Lista RezervisaniBrojevi je posebno važna za enterprise zahtev vezan za jedinstvene delovodne brojeve.

Identifikovana polja:
Internal name

Display name

Tip

Title

Title

Single line of text

RezervisaniBroj

Rezervisani broj

Number

DatumRezervacije

DatumRezervacije

Date and Time

ID

ID

Counter

Created

Created

Date and Time

Modified

Modified

Date and Time

Author

Created By

Person or Group

Editor

Modified By

Person or Group

Zaključak:

Lista najverovatnije služi za rezervaciju brojeva pre finalnog kreiranja dokumenta ili predmeta.

Status: PRETPOSTAVKA
Razlog: naziv liste i polja ukazuju na rezervaciju brojeva, ali nije dostavljena logika flow-a koji koristi ovu listu.

⸻

7.3 Shared Documents

Biblioteka Shared Documents je glavna biblioteka dokumenata.

Identifikovana polja:
Internal name

Display name

Tip

FileLeafRef

Name

File

Title

Title

Single line of text

_ExtendedDescription

Description

Multiple lines of text

DelovodniBroj

DelovodniBroj

Single line of text

EdokumentID

EdokumentID

Single line of text

Edokument

Edokument

Yes/No

Attachment

Attachment

Yes/No

MediaServiceImageTags

Image Tags

Managed Metadata

ContentType

Content Type

Computed

Zaključak:

Ova biblioteka čuva dokumente koji se zavode u sistemu i povezuje ih sa delovodnim brojem.

⸻

7.4 EmailDocuments

Biblioteka EmailDocuments je posebna biblioteka za dokumente dobijene ili kreirane iz email procesa.

Identifikovana polja:
Internal name

Display name

Tip

FileLeafRef

Name

File

Title

Title

Single line of text

_ExtendedDescription

Description

Multiple lines of text

PosiljalacEmail

PosiljalacEmail

Single line of text

Posiljalac

Posiljalac

Single line of text

MediaServiceImageTags

Image Tags

Managed Metadata

ContentType

Content Type

Zaključak:

Ova biblioteka verovatno služi kao ulazni kanal za dokumente pristigle emailom.

Status: PRETPOSTAVKA
Razlog: naziv biblioteke i polja PosiljalacEmail i Posiljalac ukazuju na email intake proces, ali nije dostavljen flow.

⸻
7.5 Exports

Biblioteka Exports služi za generisane fajlove i exporte.

Identifikovana polja:
Internal name

Display name

Tip

FileLeafRef

Name

File

Title

Title

Single line of text

_ExtendedDescription

Description

Multiple lines of text

ContentType

Content Type

Computed

Zaključak:

Biblioteka je verovatno namenjena za generisane izveštaje, exporte ili dokumentaciju koju sistem proizvodi.

Status: PRETPOSTAVKA

⸻

8. Ključni rizici

8.1 Rizik duplog delovodnog broja

Najveći rizik sistema je situacija u kojoj dva korisnika istovremeno zavode dokument i dobiju isti delovodni broj.

Ovo mora biti rešeno na nivou backend logike, ne samo na nivou Power Apps forme.

Obavezna zaštita:
Atomic counter update

Optimistic concurrency

Retry policy

Unique constraint

Audit log

Controlled error response

8.2 Rizik od logike u Canvas aplikaciji

Ako Canvas aplikacija sama računa sledeći delovodni broj, postoji rizik od race condition problema.

Canvas aplikacija sme da:

* prikaže formu
* validira obavezna polja
* pozove flow
* prikaže rezultat

Canvas aplikacija ne sme samostalno da:

* određuje konačni delovodni broj
* garantuje jedinstvenost broja
* vrši finalnu rezervaciju bez server-side potvrde

⸻

8.3 Rizik od SharePoint ograničenja

SharePoint Online može biti dobar data layer za manja i srednja rešenja, ali kod enterprise pisarnice treba obratiti pažnju na:

* delegaciju
* list view threshold
* indexing
* unique constraints
* item-level permissions
* audit
* race conditions
* retry i throttling
* flow timeout
* skalabilnost

⸻

9. Predlog strukture GitHub repozitorijuma

doccentral-v6-documentation/
│
├── README.md
│
├── docs/
│   ├── 01-business-overview.md
│   ├── 02-current-architecture.md
│   ├── 03-sharepoint-data-model.md
│   ├── 04-appconfig-analysis.md
│   ├── 05-document-libraries.md
│   ├── 06-numbering-and-concurrency.md
│   ├── 07-power-apps-analysis.md
│   ├── 08-power-automate-analysis.md
│   ├── 09-security-permissions.md
│   ├── 10-known-issues-technical-debt.md
│   ├── 11-enterprise-improvements.md
│   └── 12-cloud-code-development-brief.md
│
├── architecture/
│   ├── current-architecture.md
│   ├── target-architecture.md
│   └── numbering-sequence.md
│
├── data-model/
│   ├── appconfig.md
│   ├── rezervisani-brojevi.md
│   ├── shared-documents.md
│   ├── email-documents.md
│   └── exports.md
│
├── prompts/
│   └── cloud-code-master-prompt.md
│
└── backlog/
    ├── known-issues.md
    ├── enterprise-backlog.md
    └── open-questions.md


