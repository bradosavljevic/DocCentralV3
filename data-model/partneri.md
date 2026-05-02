# SharePoint lista: Partneri

## 1. Svrha liste

Lista **Partneri** predstavlja centralni šifarnik poslovnih partnera u rešenju **DocCentral**.

Koristi se za povezivanje dokumenata, faktura i predmeta sa poslovnim partnerima. Partner može biti kreiran ili ažuriran na osnovu podataka iz SAP integracije.

## 2. Uloga u rešenju

**Poznato:**

- Partneri se koriste kao lookup iz drugih lista, posebno kod evidencije faktura i predmeta.
- Postoji integracija iz SAP-a koja pri uvozu fakture proverava da li partner već postoji.
- Provera se trenutno radi po PIB polju.
- Ako PIB ne postoji, partner se kreira.
- Ako PIB postoji, trenutna logika ga preskače.
- Klijent traži doradu da se postojeći partner automatski ažurira ako se u SAP-u promene podaci kao što su naziv, grad ili adresa.

## 3. Ključne kolone

| Display name | Interni naziv | Tip | Poznato / Nepoznato | Opis |
|---|---|---|---|---|
| Title | Title | Single line of text | Poznato | Interni naziv / oznaka stavke. |
| PoslovnoIme | PoslovnoIme | Single line of text | Poznato | Poslovno ime partnera. |
| PIB | NEPOZNATO | Single line of text / Number | Poznato po funkciji | Poreski identifikacioni broj. Koristi se kao ključ za proveru postojanja partnera. |
| MB | MB | Single line of text | Poznato | Matični broj partnera. |
| Mesto | Mesto | Single line of text | Poznato | Grad / mesto partnera. |
| Adresa | Adresa | Single line of text | Poznato | Adresa partnera. |
| Grad | NEPOZNATO | Single line of text | Delimično poznato | Može biti posebno polje ili isto kao Mesto. |
| SAP ID | NEPOZNATO | Single line of text / Number | Nepoznato | Identifikator iz SAP-a, ako postoji. |
| Modified | Modified | Date and Time | Sistemsko | Datum poslednje izmene. |
| Created | Created | Date and Time | Sistemsko | Datum kreiranja. |
| Author | Author | Person | Sistemsko | Kreirao. |
| Editor | Editor | Person | Sistemsko | Izmenio. |

## 4. Primer JSON odgovora ka Power Apps

Poznat primer strukture koju Power Automate vraća ka Power Apps aplikaciji:

```json
[
  {
    "ID": 1,
    "Title": "PP - 1",
    "MB": "66439089",
    "Mesto": "Lazarevac",
    "Adresa": "Ibarski put 29",
    "PoslovnoIme": "Marko Milovanović PR postavljanje behaton ploča HANI 2018"
  }
]
```

## 5. Poznata integraciona logika

### Trenutno stanje

1. SAP šalje podatke o fakturi.
2. Flow proverava da li partner postoji u listi **Partneri**.
3. Provera se radi po PIB-u.
4. Ako partner ne postoji, kreira se nova stavka.
5. Ako partner postoji, flow ga preskače.

### Zahtevana dorada

Za postojeće partnere koji su iz SAP-a uneti u DocCentral, potrebno je automatski proveriti i ažurirati podatke ako se promene u SAP-u.

Minimalna polja za poređenje:

- PoslovnoIme / Naziv
- Grad / Mesto
- Adresa
- PIB
- MB, ako je dostupan

## 6. Predložena update logika

1. Normalizovati vrednosti iz SAP-a i SharePoint-a:
   - trim;
   - case-insensitive poređenje gde je poslovno prihvatljivo;
   - uklanjanje viška razmaka;
   - pravilno tretiranje null / empty vrednosti.
2. Uraditi `Get items` nad listom Partneri sa filterom po PIB-u.
3. Ako nije pronađen partner:
   - kreirati novu stavku.
4. Ako je pronađen tačno jedan partner:
   - uporediti SAP vrednosti sa SharePoint vrednostima;
   - ako postoji razlika, ažurirati samo promenjena polja;
   - upisati audit zapis.
5. Ako je pronađeno više partnera sa istim PIB-om:
   - ne raditi automatski update;
   - upisati grešku u log;
   - označiti za ručnu proveru.

## 7. Preporuke za kvalitet podataka

- PIB treba da bude unique gde god je moguće.
- Ako SharePoint ne može bezbedno da garantuje unique kroz poslovna pravila, treba dodati kontrolni flow.
- Potrebno je sprečiti duplikate po PIB-u.
- Potrebno je voditi audit za izmene koje dolaze iz SAP-a.
- Potrebno je razlikovati ručno kreirane partnere i SAP partnere, ako poslovni proces to zahteva.

## 8. Poznate nepoznanice

- Tačan interni naziv PIB polja.
- Da li je PIB kolona tekst ili broj.
- Da li je PIB unique.
- Da li postoji SAP ID polje.
- Da li postoji flag koji označava da je partner došao iz SAP-a.
- Da li postoje obavezna polja.
- Da li postoji validacija nad PIB / MB formatom.
- Da li postoje lookup veze iz više lista, osim faktura / predmeta.

## 9. Preporuka za dalju dokumentaciju

Dopuniti dokument iz SharePoint REST exporta liste **Partneri** sledećim vrednostima:

- sve kolone;
- interni nazivi;
- tipovi;
- required / hidden / read only;
- indexed;
- enforce unique;
- default values;
- lookup veze;
- permission model;
- versioning podešavanja.
