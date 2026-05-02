# 07 — Power Apps Canvas aplikacija

## 1. Svrha dokumenta

Ovaj dokument opisuje trenutno poznatu strukturu i ulogu Power Apps Canvas aplikacije u rešenju **DocCentral v6.0**.

Cilj dokumenta je da se zabeleži:

- šta je potvrđeno o aplikaciji
- koje funkcionalnosti aplikacija pokriva
- kako aplikacija komunicira sa Power Automate flow-ovima
- šta aplikacija ne sme da radi zbog rizika od race condition problema
- koje informacije još nisu poznate i treba ih dopuniti kroz dalju analizu solution-a

---

## 2. Status analize

| Oblast | Status |
|---|---|
| Tip aplikacije | POTVRĐENO |
| Početni ekran | POTVRĐENO |
| Ekrani aplikacije | DELIMIČNO POZNATO |
| Power Fx logika | NEPOZNATO |
| Kolekcije | NEPOZNATO |
| Flow integracije | DELIMIČNO POZNATO |
| Zavođenje dokumenta | DELIMIČNO POZNATO |
| Pregled dokumenata | POTVRĐENO |
| Email dokumenti | POTVRĐENO na nivou funkcionalnosti |
| Export | POTVRĐENO na nivou funkcionalnosti |
| Administracija / konfiguracija | POTVRĐENO na nivou koncepta |

---

## 3. Tip aplikacije

DocCentral v6.0 koristi **Power Apps Canvas App**.

Ovo nije SharePoint customized form, već posebna Canvas aplikacija.

### Činjenice

- Tip aplikacije: `Canvas App`
- Aplikacija koristi SharePoint Online kao data/storage layer.
- Aplikacija koristi Power Automate flow-ove za određene backend operacije.
- Za zavođenje dokumenta koristi se poseban Power Automate flow zbog rizika od race condition problema.

---

## 4. Početni ekran

Glavni početni ekran aplikacije je:

```text
scrHome
```

### Zaključak

`scrHome` predstavlja početni ekran / home screen aplikacije.

### Status

POTVRĐENO.

---

## 5. Poznate funkcionalnosti aplikacije

Na osnovu dosadašnjih informacija, aplikacija pokriva sledeće funkcionalnosti:

| Funkcionalnost | Status | Napomena |
|---|---:|---|
| Početni ekran | POTVRĐENO | `scrHome` |
| Zavođenje dokumenta | POTVRĐENO | Postoji poseban ekran/funkcionalnost |
| Pregled dokumenata | POTVRĐENO | Ne postoji poseban ekran; pregled je direktno u SharePoint listi |
| Email dokumenti | POTVRĐENO | Power Automate čita shared mailbox i zavodi dokumente iz mailova sa attachmentom |
| Export | POTVRĐENO | Export šifarnika i konfiguracije iz `AppConfig` |
| Administracija / konfiguracija | POTVRĐENO | Šifarnici i JSON konfiguracije nalaze se u `AppConfig` listi |

---

## 6. Zavođenje dokumenta

Zavođenje dokumenta postoji kao posebna funkcionalnost u Canvas aplikaciji.

### Bitna arhitektonska napomena

Zavođenje dokumenta ne sme da se oslanja samo na Power Apps logiku za generisanje konačnog delovodnog broja.

Razlog:

- više korisnika može istovremeno da zavodi dokumente
- sistem ne sme nikada da dozvoli da dva dokumenta dobiju isti delovodni broj
- race condition problem je već prepoznat kao kritičan rizik

### Backend flow za delovodni broj

Za rešavanje problema race condition-a koristi se poseban Power Automate flow:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

### Zaključak

Canvas aplikacija treba da pozove backend flow, a konačna dodela delovodnog broja mora biti potvrđena kroz backend logiku.

---

## 7. Upload dokumenta

Upload dokumenta ide preko posebnog Power Automate flow-a:

```text
Upload Doc
```

### Činjenice

- Dokument se ne uploaduje isključivo direktnom Power Apps Patch logikom.
- Za upload postoji namenski flow.
- Flow `Upload Doc` je deo procesa rada sa dokumentima.

### Status

POTVRĐENO na nivou naziva i namene flow-a.

### Nepoznato

- tačan trigger flow-a
- ulazni parametri
- izlazni response prema Canvas aplikaciji
- tačna SharePoint biblioteka u koju upisuje dokument
- da li radi metadata update odmah nakon upload-a
- da li koristi service account konekciju

---

## 8. Pregled dokumenata

Za pregled dokumenata ne postoji poseban ekran u Canvas aplikaciji.

Pregled se radi direktno u SharePoint listi / biblioteci.

### Zaključak

Canvas aplikacija nije glavni UI za pretragu i pregled svih dokumenata.

Ovo znači da SharePoint view-ovi, kolone, filteri, indeksiranje i permissions model imaju važnu ulogu u korisničkom iskustvu.

### Status

POTVRĐENO.

---

## 9. Email dokumenti

Email dokumenti su posebna funkcionalnost.

Opis procesa:

```text
Shared mailbox
        |
        v
Power Automate flow
        |
        v
Čitanje emailova sa attachmentima
        |
        v
Zavođenje / upis dokumenata u DocCentral
```

### Činjenice

- Power Automate čita shared mailbox.
- Dokumenti iz emailova sa attachmentom se zavode kroz sistem.
- Postoji SharePoint biblioteka `EmailDocuments`.

### Identifikovana SharePoint biblioteka

```text
EmailDocuments
```

### Poznata polja biblioteke `EmailDocuments`

| Internal name | Display name | Tip |
|---|---|---|
| FileLeafRef | Name | File |
| Title | Title | Single line of text |
| _ExtendedDescription | Description | Multiple lines of text |
| PosiljalacEmail | PosiljalacEmail | Single line of text |
| Posiljalac | Posiljalac | Single line of text |
| MediaServiceImageTags | Image Tags | Managed Metadata |
| ContentType | Content Type | Computed |

### Zaključak

Biblioteka `EmailDocuments` najverovatnije služi kao ulazna ili procesna biblioteka za dokumente koji dolaze iz email procesa.

### Status

DELIMIČNO POTVRĐENO.

---

## 10. Export funkcionalnost

Export funkcionalnost služi za export šifarnika i konfiguracije iz liste `AppConfig`.

### Činjenice

- Postoji biblioteka `Exports`.
- Export obuhvata šifarnike i konfiguracije iz `AppConfig`.
- `AppConfig` sadrži JSON konfiguracije i šifarnike.

### Identifikovana SharePoint biblioteka

```text
Exports
```

### Poznata polja biblioteke `Exports`

| Internal name | Display name | Tip |
|---|---|---|
| FileLeafRef | Name | File |
| Title | Title | Single line of text |
| _ExtendedDescription | Description | Multiple lines of text |
| ContentType | Content Type | Computed |

### Status

POTVRĐENO na nivou namene.

---

## 11. Administracija i konfiguracija

Svi šifarnici i konfiguracioni JSON podaci nalaze se u SharePoint listi:

```text
AppConfig
```

### Poznata polja liste `AppConfig`

| Internal name | Display name | Tip |
|---|---|---|
| Title | Title | Single line of text |
| Config | Config | Multiple lines of text |
| ColumnHeader | ColumnHeader | Multiple lines of text |
| ID | ID | Counter |
| Created | Created | Date and Time |
| Modified | Modified | Date and Time |
| Author | Created By | Person or Group |
| Editor | Modified By | Person or Group |

### Pretpostavka

Canvas aplikacija verovatno učitava konfiguraciju iz `AppConfig` liste i na osnovu JSON sadržaja puni kolekcije, šifarnike, prevode i pravila ponašanja aplikacije.

### Status

DELIMIČNO POTVRĐENO.

---

## 12. Uloga Canvas aplikacije u arhitekturi

Canvas aplikacija treba da bude frontend sloj.

Preporučena odgovornost Canvas aplikacije:

- prikaz početnog ekrana
- prikaz forme za zavođenje dokumenta
- unos i validacija osnovnih korisničkih podataka
- pozivanje Power Automate flow-ova
- prikaz rezultata korisniku
- prikaz grešaka korisniku
- navigacija kroz funkcionalnosti

Canvas aplikacija ne treba da bude odgovorna za:

- konačno generisanje delovodnog broja
- garantovanje jedinstvenosti delovodnog broja
- konkurentno zaključavanje broja
- direktno rešavanje race condition problema
- kompleksnu backend validaciju
- audit logiku
- sigurnosno-kritične operacije
- dodelu prava na SharePoint objektima

---

## 13. Kritičan zahtev: jedinstven delovodni broj

Najvažniji zahtev za aplikaciju i backend jeste:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

### Posledice po Canvas aplikaciju

Canvas aplikacija mora da tretira delovodni broj kao rezultat backend procesa, a ne kao lokalno izračunatu vrednost.

Ispravan obrazac:

```text
Korisnik popunjava formu
        |
        v
Canvas App poziva Power Automate flow
        |
        v
Flow rezerviše / dodeljuje delovodni broj uz concurrency kontrolu
        |
        v
Flow vraća rezultat Canvas aplikaciji
        |
        v
Canvas App prikazuje uspeh ili grešku
```

Neispravan obrazac:

```text
Canvas App pročita poslednji broj
        |
        v
Canvas App lokalno uveća broj za 1
        |
        v
Canvas App upiše dokument
```

Ovaj obrazac nije bezbedan za istovremeni rad više korisnika.

---

## 14. Poznati Power Automate flow-ovi povezani sa aplikacijom

| Flow | Poznata namena | Status |
|---|---|---|
| `Upload Doc` | Upload dokumenta | POTVRĐENO |
| `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps` | Dodela / upis delovodnog broja za novi dokument iz Power Apps | POTVRĐENO |
| Email intake flow | Čitanje shared mailbox-a i obrada attachmenta | POSTOJI, naziv NEPOZNAT |
| Export flow | Export šifarnika i konfiguracije iz AppConfig | POSTOJI, naziv NEPOZNAT |

---

## 15. Poznati SharePoint elementi koje koristi aplikacija

| Element | Tip | Namena |
|---|---|---|
| `AppConfig` | SharePoint lista | Šifarnici, JSON konfiguracije, administracija |
| `RezervisaniBrojevi` | SharePoint lista | Rezervacija / kontrola brojeva |
| `Shared Documents` | Document library | Glavna biblioteka dokumenata |
| `EmailDocuments` | Document library | Dokumenti iz email procesa |
| `Exports` | Document library | Exportovani fajlovi |

---

## 16. Nepoznato / potrebno dopuniti

Sledeće informacije još nisu poznate i treba ih dopuniti nakon pregleda Canvas app source-a ili dodatnih screenshotova:

1. Tačan naziv ekrana za zavođenje dokumenta.
2. Da li postoje dodatni ekrani osim `scrHome`.
3. Power Fx kod za `App.OnStart`.
4. Power Fx kod za `scrHome.OnVisible`.
5. Sve globalne varijable (`Set(...)`).
6. Sve lokalne varijable (`UpdateContext(...)`).
7. Sve kolekcije (`ClearCollect(...)`, `Collect(...)`).
8. Način učitavanja `AppConfig` konfiguracije.
9. Tačan način pozivanja flow-a `Upload Doc`.
10. Tačan način pozivanja flow-a `CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps`.
11. Response schema flow-ova prema Canvas aplikaciji.
12. Da li aplikacija koristi direktan Patch prema SharePoint-u.
13. Da li aplikacija koristi Power Automate za sve Create/Edit/Delete operacije.
14. Da li korisnici imaju Read Only prava nad SharePoint-om.
15. Da li se svi upisi izvršavaju preko service account konekcija.
16. Koja polja su visible/editable na formama.
17. Koja polja su obavezna za zavođenje.
18. Da li postoji višejezičnost.
19. Da li postoji responzivni layout.
20. Da li postoje role-based UI kontrole.

---

## 17. Preporuke za novu verziju Canvas aplikacije

Za novu enterprise verziju aplikacije preporučuje se:

### 17.1 Frontend odgovornosti

Canvas aplikacija treba da bude lagan, responzivan frontend.

Treba da radi:

- unos podataka
- lokalnu validaciju
- pozivanje backend flow-ova
- prikaz statusa
- prikaz grešaka
- navigaciju

Ne treba da radi:

- generisanje delovodnog broja
- direktne sigurnosno-kritične upise
- kompleksne backend odluke
- permission logiku

---

### 17.2 Responsiveness

Nova aplikacija treba da bude potpuno responzivna.

Obavezno:

- koristiti responsive containers
- izbegavati fiksne X/Y pozicije gde nije neophodno
- podržati desktop, tablet i širi browser prikaz
- definisati layout breakpoints

---

### 17.3 Višejezičnost

Nova aplikacija treba da podrži najmanje:

- srpski
- engleski

Preporučeni pristup:

- prevode držati u `AppConfig`
- učitati prevode u kolekciju pri startu aplikacije
- koristiti centralnu funkciju / lookup za prikaz labela
- ne hardkodovati tekstove u kontrolama

---

### 17.4 Performanse

Preporuke:

- minimizovati broj direktnih SharePoint poziva iz aplikacije
- učitavati šifarnike kontrolisano
- koristiti Power Automate za složenija čitanja i transformacije
- koristiti indeksirane kolone u SharePoint listama
- izbegavati nedelegabilne filtere nad velikim listama
- koristiti keširanje konfiguracije kada je moguće

---

### 17.5 Sigurnost

Preporuke:

- korisnici ne treba da imaju direktna edit prava nad ključnim SharePoint listama ako poslovna logika zahteva kontrolisane upise
- Create/Edit/Delete operacije treba da idu preko Power Automate flow-ova
- flow-ovi treba da rade preko kontrolisane konekcije / service account-a
- svaka kritična operacija treba da ima audit zapis
- greške treba zapisivati u posebnu log listu

---

## 18. Zaključak

Power Apps Canvas aplikacija u DocCentral v6.0 ima ulogu korisničkog interfejsa za početni rad i zavođenje dokumenata, dok kritične backend operacije treba da budu poverene Power Automate flow-ovima.

Najvažniji deo arhitekture je kontrola delovodnog broja.

Canvas aplikacija ne sme biti mesto gde se garantuje jedinstvenost delovodnog broja. Ta odgovornost mora biti u backend logici, kroz flow kao što je:

```text
CF_DocCentral21_AddWorkbookNumberToNewDocPowerApps
```

Za kompletnu analizu potrebno je dopuniti dokument stvarnim Power Fx kodom iz aplikacije, nazivima svih ekrana, kolekcija, varijabli i tačnim flow response šemama.

