# Claude Code - DocCentral

Ovaj folder sadrži razvojne instrukcije, domenska pravila i skillove za Claude Code za projekat **DocCentral / e-pisarnica**.

Glavni folder: `claude-code/skills`.

Claude Code mora prvo pročitati ovaj fajl, zatim `claude-code/skills/README.md`, a nakon toga domenske skillove koji odgovaraju konkretnom zadatku.

---

## 1. Svrha ovog foldera

Folder `claude-code` nije izvorni kod postojeće aplikacije, već razvojni i dokumentacioni paket koji Claude Code koristi kao osnovu za:

- razumevanje poslovnog procesa DocCentral / e-pisarnica;
- razvoj nove unapređene verzije Canvas aplikacije;
- dizajn Power Automate backend sloja;
- projektovanje SharePoint data modela;
- projektovanje sigurnosnog i permission modela;
- izradu tehničke dokumentacije spremne za GitHub;
- održavanje standarda razvoja kroz posebne skill fajlove.

Ovaj paket treba tretirati kao **radni razvojni standard projekta**, a ne kao finalni opis svih implementacionih detalja.

---

## 2. Kontekst projekta

DocCentral je Power Platform rešenje za elektronsku pisarnicu / zavođenje i arhiviranje dokumenata.

Rešenje se zasniva na sledećem obrascu:

- **Power Apps Canvas aplikacija** kao korisnički interfejs;
- **Power Automate flow-ovi** kao backend sloj za upis, obradu i kontrolisane operacije;
- **SharePoint Online liste i biblioteke** kao primarni data storage;
- **App Config lista** kao izvor šifarnika, konfiguracija i pravila koje aplikacija čita;
- **Entra ID / Microsoft 365 grupe** kao osnova za organizacioni i sigurnosni model;
- **servisni nalog** kao izvršilac backend upisa preko Power Automate-a.

Cilj nove verzije je enterprise-ready rešenje sa jasnim razdvajanjem UI sloja, backend logike, data modela, sigurnosti, audit/logging sloja i deployment procedure.

---

## 3. Najvažnija arhitektonska pravila

### 3.1 SharePoint prava

Korisnici treba da imaju **Read Only** prava nad SharePoint podacima.

Svi upisi, izmene, brisanja, rezervacije brojeva i sistemske operacije moraju ići kroz Power Automate flow-ove koji rade pod servisnim nalogom.

Canvas aplikacija ne sme direktno da radi kritične `Patch`, `Remove`, `SubmitForm` ili slične operacije nad SharePoint listama ako korisnik nema pravo upisa.

### 3.2 Backend preko Power Automate-a

Power Automate je backend sloj rešenja.

Flow-ovi moraju biti projektovani kao kontrolisane operacije:

- validacija ulaza;
- provera prava;
- upis u SharePoint;
- audit zapis;
- greška sa jasnom porukom;
- standardizovan response prema Power Apps aplikaciji.

### 3.3 Delovodni broj

Delovodni broj je kritičan enterprise zahtev.

Sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Za novu verziju obavezno projektovati concurrency-safe mehanizam koji uključuje:

- centralni izvor sledećeg broja;
- rezervaciju broja;
- jedinstveni constraint na `DelovodniBroj`;
- retry logiku;
- audit neuspelih pokušaja;
- jasnu grešku ako rezervacija ne uspe;
- zaključavanje delovodnika po godini;
- arhivsku knjigu za završenu godinu.

Trenutno poznato:

- App Config sadrži konfiguraciju **Delovodne knjige**;
- u konfiguraciji se čuva sledeći delovodni broj;
- postoji lista **Rezervisani Brojevi**;
- `DelovodniBroj` je unikatan;
- ETag se trenutno ne koristi;
- retry logika trenutno nije implementirana, ali je obavezna za novu verziju;
- audit neuspelih pokušaja trenutno nije implementiran, ali ga treba dodati.

### 3.4 Sigurnost nije samo UI filtriranje

Power Apps može da sakrije kontrole, ekrane i podatke, ali to nije dovoljno.

Sigurnost mora postojati i na backend sloju:

- item-level permissions;
- break inheritance kada je potrebno;
- Read prava za korisnike/grupe;
- RW prava za servisni nalog i vlasnike;
- SharePoint mora fizički sprečiti pristup dokumentima koje korisnik ne treba da vidi.

### 3.5 App Config driven development

Aplikacija treba da koristi App Config za šifarnike i konfiguracije.

Konfiguracije ne smeju biti hardkodovane u kontrolama ako mogu biti pročitane iz App Config liste.

App Config treba koristiti za:

- tipove dokumenata;
- delovodne knjige;
- statuse;
- konfiguraciju prikaza;
- šifarnike;
- pravila ponašanja aplikacije;
- jezičke ili tekstualne resurse ako postoje.

---

## 4. Poznati moduli i domeni

### 4.1 Svi predmeti

Lista **Svi predmeti** predstavlja centralni registar predmeta/dokumenata.

Koristi se za evidenciju zavednih, arhiviranih i drugih dokumenata u procesu.

Važno:

- mora imati stabilan identifikator dokumenta;
- mora podržati jedinstveni delovodni broj;
- mora podržati status dokumenta;
- mora podržati organizacionu jedinicu / sigurnosni kontekst;
- mora biti povezana sa dokument bibliotekama;
- mora imati audit/logging podršku u novoj verziji.

### 4.2 Partneri

Lista **Partneri** sadrži poslovne partnere.

Poznato:

- partner se najčešće proverava po PIB-u;
- postojeći flow proverava da li partner postoji;
- ako PIB ne postoji, partner se kreira;
- nova dorada treba da proverava promene podataka kao što su naziv, grad i adresa;
- ako se podaci razlikuju od izvora, postojeći partner treba da se ažurira.

### 4.3 App Config

Lista **App Config** sadrži šifarnike i konfiguracije koje aplikacija čita.

XML export lista služi za razumevanje schema i kolona.

JSON u App Config-u predstavlja konfiguracione objekte i šifarnike koje aplikacija koristi u runtime-u.

### 4.4 Rezervisani Brojevi

Lista **Rezervisani Brojevi** postoji i treba da se koristi ili unapredi za mehanizam rezervacije delovodnog broja.

Za novu verziju ova lista treba da bude deo concurrency-safe dizajna.

### 4.5 Dokument biblioteke

Dokument biblioteke čuvaju fajlove vezane za predmete.

Pristup dokumentima mora biti kontrolisan SharePoint permisijama, ne samo filtriranjem u Power Apps aplikaciji.

---

## 5. Power Apps pravila za novu verziju

Claude Code treba da projektuje Canvas aplikaciju na osnovu funkcionalnih zahteva, a ne da mehanički kopira postojeći broj ekrana.

### Claude Code sme sam da odluči

Claude Code može sam da predloži i implementira:

- broj ekrana;
- nazive ekrana;
- strukturu ekrana;
- kontrole po ekranima;
- navigacioni tok;
- raspored komponenti;
- responsive layout;
- UX optimizacije.

### Claude Code mora da pročita iz solution-a

Iz postojećeg solution exporta treba pročitati i dokumentovati:

- `OnStart`;
- `App.Formulas`;
- `OnVisible` formule;
- `OnSelect` formule;
- `OnChange` formule;
- `OnSave` logiku;
- kolekcije;
- globalne promenljive;
- lokalne promenljive;
- pozive flow-ova;
- izvore podataka;
- postojeći UX tok za unos, pregled, izmenu i arhiviranje.

### Obavezna pravila za novu Canvas aplikaciju

Nova aplikacija mora biti:

- potpuno responzivna;
- brza;
- pregledna;
- višejezična: srpski i engleski;
- spremna za enterprise usage;
- odvojena od direktnih SharePoint write operacija;
- zasnovana na konfiguracijama gde god je to realno;
- kompatibilna sa read-only SharePoint korisničkim pravima.

---

## 6. Power Automate pravila za novu verziju

Claude Code treba da tretira Power Automate kao backend API sloj.

Iz postojećeg solution-a treba pročitati:

- kompletan spisak flow-ova;
- tačne nazive flow-ova;
- trigger-e;
- akcije po koracima;
- konektore;
- connection reference;
- environment variables;
- child flow zavisnosti;
- retry i error handling logiku;
- owner/run-only podešavanja ako su dostupna;
- koji flow čita SharePoint;
- koji flow piše u SharePoint;
- koji flow se poziva iz Power Apps aplikacije;
- koji flow se pokreće preko SharePoint trigger-a;
- koji flow radi SAP/external import ako je deo solution-a;
- koji flow radi partner sync;
- koji flow generiše ili rezerviše delovodni broj.

Za novu verziju svaki važan flow treba da ima:

- standardizovan ulazni JSON;
- validaciju ulaza;
- kontrolu prava;
- centralizovan error handling;
- retry politiku gde ima smisla;
- audit log;
- standardizovan izlaz prema Power Apps aplikaciji.

---

## 7. Security / permission model

Poznato:

- korisnici imaju Read Only nad SharePoint-om;
- servisni nalog ima RW prava;
- upisi idu kroz Power Automate;
- pristup dokumentima zavisi od organizacionih jedinica;
- Entra ID / M365 grupe se koriste za prava;
- owner ima RW;
- members/viewers imaju Read;
- nema posebne admin grupe kao obaveznog globalnog modela;
- pristup mora biti sproveden i u Canvas aplikaciji i u SharePoint-u;
- SharePoint item/folder/file permisije se koriste za stvarno skrivanje sadržaja.

Nepoznato / zavisi od klijenta:

- kompletna mapa Entra ID grupa;
- koje grupe postoje kod konkretnog klijenta;
- koje organizacione jedinice vidi koja grupa;
- da li klijent želi posebne role izvan organizacionih jedinica.

Claude Code ne sme pretpostaviti konkretne grupe ako nisu definisane.

---

## 8. Šta se za sada ignoriše

SAP / eksterni import se za sada ignoriše osim ako je direktno potreban za razumevanje postojećeg solution-a.

Ne treba projektovati SAP integraciju dok korisnik izričito ne traži taj deo.

---

## 9. Poznate nepoznanice

Ove teme treba dokumentovati kao `NEPOZNATO` ako nisu vidljive iz solution-a ili dostavljene dokumentacije:

- kompletna lista aktivnih App Config ključeva;
- koji ekrani koriste koje App Config vrednosti;
- kompletan approval proces;
- da li je approval sekvencijalan ili paralelan;
- gde se čuvaju approval odgovori;
- da li postoji bulk approval;
- PDF generisanje;
- Word/HTML template logika;
- verzionisanje finalnih PDF-ova;
- centralni logging;
- audit model;
- error queue / dead-letter lista;
- svi indexed columns;
- svi content types;
- SharePoint views;
- validation settings;
- hidden/system kolone koje aplikacija koristi.

Ako Claude Code nema pouzdan podatak, mora napisati: `NEPOZNATO`.

Ne sme izmišljati detalje.

---

## 10. Struktura skillova

Folder `claude-code/skills` sadrži domenske instrukcije.

Trenutni skillovi:

| Fajl | Namena |
|---|---|
| `project-context.md` | Glavni projektni kontekst |
| `power-apps-canvas-development.md` | Canvas app razvojna pravila |
| `power-apps-responsive-ui.md` | Responsive UI pravila |
| `power-apps-form-patterns.md` | Pravila za forme i unos podataka |
| `power-apps-localization.md` | Srpski/engleski i lokalizacija |
| `power-apps-security-filtering.md` | UI filtriranje i sigurnosna pravila |
| `power-automate-backend-patterns.md` | Backend flow obrasci |
| `power-automate-sharepoint-write-pattern.md` | SharePoint write preko flow-ova |
| `power-automate-error-handling.md` | Error handling standard |
| `power-automate-logging-audit.md` | Logging i audit standard |
| `power-automate-child-flow-patterns.md` | Child flow standard |
| `sharepoint-data-modeling.md` | SharePoint modeliranje |
| `sharepoint-permissions.md` | SharePoint permission model |
| `sharepoint-item-level-security.md` | Item-level security |
| `sharepoint-document-libraries.md` | Biblioteke dokumenata |
| `delovodni-broj-concurrency.md` | Kritičan model delovodnog broja |
| `app-config-driven-development.md` | App Config development |
| `document-lifecycle.md` | Životni ciklus dokumenta |
| `partner-management.md` | Partneri i sync logika |
| `approval-process.md` | Approval proces |
| `archive-process.md` | Arhiviranje i zaključavanje godine |
| `github-documentation-style.md` | Stil dokumentacije za GitHub |
| `coding-standards.md` | Kodni standardi |
| `testing-strategy.md` | Test strategija |
| `deployment-checklist.md` | Deployment checklist |

---

## 11. Kako Claude Code treba da koristi ovaj paket

Za svaki zadatak Claude Code treba da uradi sledeće:

1. Pročita `claude-code/README.md`.
2. Pročita `claude-code/skills/README.md`.
3. Izabere relevantne skill fajlove.
4. Proveri da li zadatak zahteva čitanje postojećeg solution-a.
5. Razdvoji:
   - činjenice;
   - pretpostavke;
   - nepoznato.
6. Ne izmišlja nazive lista, kolona, flow-ova, ekrana ili grupa.
7. Ako nešto nije vidljivo, označi kao `NEPOZNATO`.
8. Predlaže enterprise unapređenja odvojeno od postojećeg stanja.

---

## 12. Obavezni stil dokumentacije

Sva dokumentacija mora biti:

- na srpskom jeziku;
- tehnička;
- jasna;
- GitHub-ready;
- strukturisana kroz `.md` fajlove;
- bez marketing narativa;
- bez izmišljanja;
- sa jasnim razdvajanjem postojećeg stanja i predloga za novu verziju.

Preporučena struktura svakog većeg dokumenta:

```md
# Naziv dokumenta

## Svrha

## Postojeće stanje

## Poznate činjenice

## Pretpostavke

## Nepoznato

## Pravila za novu verziju

## Tehničke preporuke

## Rizici

## Checklist
```

---

## 13. Pravila za novu verziju rešenja

Nova verzija treba da bude projektovana kao enterprise Power Platform aplikacija.

Obavezno:

- standard konektori, osim ako korisnik izričito kaže drugačije;
- Power Apps ne radi kritične upise direktno u SharePoint;
- Power Automate radi backend operacije;
- SharePoint korisnici su Read Only;
- servisni nalog izvršava write operacije;
- delovodni broj je concurrency-safe;
- App Config upravlja šifarnicima i konfiguracijama;
- security postoji i u aplikaciji i na SharePoint nivou;
- audit/logging se uvodi za kritične operacije;
- greške su jasne i korisniku i administratoru;
- dokumentacija se održava paralelno sa razvojem.

---

## 14. Kritični rizici

Najveći rizici projekta:

1. Dupliranje delovodnog broja kod istovremenog rada više korisnika.
2. Oslanjanje samo na Power Apps UI filtriranje umesto backend permissions.
3. Direktni SharePoint write iz Canvas aplikacije pod korisničkim kontekstom.
4. Nedostatak audit loga za kritične akcije.
5. Nedostatak retry/error handling mehanizma.
6. Hardkodovane vrednosti umesto App Config konfiguracije.
7. Nejasna mapa grupa i organizacionih jedinica kod konkretnog klijenta.
8. Nepotpuno dokumentovani flow-ovi i connection reference.
9. Nedokumentovan lifecycle dokumenta.
10. Nedovoljno testiran concurrent scenario za zavođenje.

---

## 15. Minimalni acceptance kriterijumi za razvoj

Nova funkcionalnost se ne smatra spremnom dok ne postoji:

- opis poslovne svrhe;
- opis tehničkog toka;
- definisan data model;
- definisana prava;
- definisan Power Automate backend tok ako ima upis;
- validacija ulaza;
- error handling;
- audit/logging gde je potrebno;
- test scenario;
- ažurirana dokumentacija.

---

## 16. GitHub radni tok

Preporučeni GitHub workflow:

```bash
git status
git add claude-code/README.md claude-code/skills/*.md
git commit -m "Update Claude Code skills and project README"
git push
```

Ako remote ima izmene:

```bash
git pull --rebase
git push
```

---

## 17. Napomena za Claude Code

Ovaj projekat ima jasna pravila:

- ne izmišljaj;
- koristi konkretne nazive iz solution-a kada postoje;
- ako nešto nije poznato, napiši `NEPOZNATO`;
- postojeće stanje i predlog nove verzije moraju biti odvojeni;
- delovodni broj tretiraj kao kritičan enterprise mehanizam;
- security nikada ne sme biti samo UI logika;
- sve write operacije moraju biti backend kontrolisane;
- dokumentacija mora ostati spremna za GitHub.
