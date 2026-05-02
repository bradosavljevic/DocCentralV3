# Cloud Code Development Brief

## Cilj

Pripremiti osnovu za razvoj nove enterprise verzije DocCentral / e-pisarnica rešenja, uz zadržavanje poslovne logike postojećeg sistema i rešavanje poznatih tehničkih ograničenja.

## Tehnološki smer

Primarni stack:
- Power Apps Canvas
- Power Automate
- SharePoint Online
- Microsoft 365 Groups
- Standard konektori, osim ako korisnik izričito odobri premium konektor

## Glavna pravila nove verzije

### 1. SharePoint permissions

Korisnici treba da imaju Read Only prava nad SharePoint listama i bibliotekama.

Sve Create, Edit i Delete operacije moraju ići preko Power Automate flow-ova.

Power Automate izvršava promene kroz service account.

### 2. Delovodni broj

Generisanje delovodnog broja mora biti concurrency-safe.

Sistem mora sprečiti duplikate čak i kada više korisnika istovremeno zavodi dokumenta.

Obavezno projektovati:
- centralizovan counter,
- locking mehanizam,
- retry logiku,
- unique constraint,
- audit log,
- error log,
- poruku korisniku u slučaju neuspeha.

### 3. Canvas app

Aplikacija mora biti:
- responzivna,
- brza,
- višejezična,
- jasna za korisnike,
- optimizovana za velike SharePoint liste,
- bez nepotrebnog direktnog pristupa upisu u SharePoint.

Jezici:
- srpski kao default,
- engleski kao dodatni jezik.

### 4. Power Automate

Flow-ovi moraju imati:
- jasan naming convention,
- child flow gde ima smisla,
- centralizovano logovanje,
- error handling,
- retry policy,
- kontrolu concurrency-ja,
- jasne response vrednosti ka Power Apps.

### 5. Audit i monitoring

Nova verzija mora imati:
- audit log listu ili tabelu,
- error log listu ili tabelu,
- zapis za svaki važan business event,
- zapis za svaku grešku,
- correlation id / run id gde je moguće,
- tehnički i business opis greške.

## Očekivani output za razvoj

Claude Code treba da pomogne u kreiranju:
- architecture dokumentacije,
- data model specifikacije,
- Power Apps screen specifikacije,
- Power Automate flow specifikacije,
- test scenarija,
- deployment checklist,
- backlog-a za novu verziju,
- Cloud Code promptova za implementaciju.
