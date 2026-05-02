# SharePoint biblioteka: Shared Documents / Dokumenta

## 1. Svrha biblioteke

Biblioteka **Shared Documents** / **Dokumenta** koristi se za čuvanje dokumenata povezanih sa predmetima u DocCentral rešenju.

U SharePoint Online rešenjima ovo je često glavna biblioteka dokumenata, ali stvarni naziv i URL mogu biti prilagođeni, npr. `Dokumenta`.

## 2. Uloga u DocCentral rešenju

**Poznato:**

- Dokumenti se povezuju sa stavkama preko delovodnog broja.
- Koristi se URL filter pattern za prikaz dokumenata po `DelovodniBroj` vrednosti.
- Poslednja verzija dokumenta u PDF formatu se arhivira u SharePoint.
- Koriste se biblioteke kao što su `Dokumenta` i potencijalno HG biblioteka.

Poznat pattern linka:

```text
.../Dokumenta/Forms/AllItems.aspx?FilterField1=DelovodniBroj&FilterValue1=<encoded value>&FilterType1=Text&viewid=<view guid>
```

## 3. Ključne metadata kolone

| Display name | Interni naziv | Tip | Poznato / Nepoznato | Opis |
|---|---|---|---|---|
| Name | FileLeafRef | File | Sistemsko | Naziv fajla. |
| Title | Title | Single line of text | Sistemsko / opciono | Naslov dokumenta. |
| DelovodniBroj | NEPOZNATO | Single line of text | Poznato po funkciji | Ključ za povezivanje sa predmetom. |
| PredmetId | NEPOZNATO | Lookup / Number | Nepoznato | Potencijalna veza ka listi Svi predmeti. |
| TipDokumenta | NEPOZNATO | Choice/Text | Delimično poznato | Tip dokumenta. |
| Status | NEPOZNATO | Choice/Text | Nepoznato | Status dokumenta. |
| LinkDoDokumenta | NEPOZNATO | Hyperlink | Poznato kao problematično polje | Korišćeno za čuvanje linka do dokumenta. |
| Modified | Modified | Date and Time | Sistemsko | Datum izmene. |
| Created | Created | Date and Time | Sistemsko | Datum kreiranja. |
| Editor | Editor | Person | Sistemsko | Izmenio. |
| Author | Author | Person | Sistemsko | Kreirao. |

## 4. Poznati tehnički problemi

### 4.1 Hyperlink polje

Poznat problem:

- Pri upisu u hyperlink kolonu, npr. `LinkDoDokumenta`, SharePoint može vratiti grešku:

```text
Invalid URL value. A URL field contains invalid data
```

Preporuka:

- proveriti da li se u hyperlink polje šalje objekat odgovarajuće strukture;
- validirati URL encoding;
- izbegavati prazan ili nevalidan URL;
- razmotriti čuvanje URL-a kao običan tekst ako nije potrebna Hyperlink kolona.

### 4.2 Permission inheritance

Poznato je da rešenje koristi dodelu prava kroz grupe / flow logiku.

Rizici:

- veliki broj item-level permission promena može usporiti SharePoint;
- masovno prekidanje nasleđivanja nad hiljadama dokumenata može biti sporo;
- potrebno je logovanje grešaka pri grant access akcijama.

## 5. Power Automate obrasci

Moguće akcije nad bibliotekom:

- create file;
- update file properties;
- get file properties;
- grant access to item or folder;
- break role inheritance preko SharePoint REST-a;
- copy file;
- move file;
- generate PDF;
- archive final PDF version.

## 6. Veze sa drugim objektima

| Objekat | Veza | Opis |
|---|---|---|
| Svi predmeti | DelovodniBroj | Dokumenti se prikazuju uz predmet. |
| Partneri | Indirektno preko predmeta | Dokument može biti povezan sa partnerom. |
| App Config | Konfiguracija | Može određivati tipove, biblioteke ili pravila. |
| Power Automate | Obrada dokumenata | Kreiranje, metadata update, prava, PDF, arhiva. |

## 7. Preporuke

- DelovodniBroj kolonu indeksirati.
- Metadata obavezno ažurirati odmah nakon upload-a.
- Ne oslanjati se samo na naziv fajla za povezivanje dokumenta.
- Uvesti audit log za svaku promenu prava.
- Uvesti error log za neuspešne grant access i update file properties akcije.
- Standardizovati URL linkove ka dokumentima.

## 8. Poznate nepoznanice

- Tačan naziv biblioteke.
- Tačan server relative URL.
- Tačan interni naziv DelovodniBroj kolone.
- Da li biblioteka ima content types.
- Da li je uključen versioning.
- Da li je uključen content approval.
- Da li postoje folderi.
- Da li postoje retention labels.
- Da li se prava dodeljuju na nivou foldera ili pojedinačnog dokumenta.
