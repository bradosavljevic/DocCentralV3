# SharePoint lista: Svi predmeti

## 1. Svrha liste

Lista **Svi predmeti** predstavlja centralnu evidenciju predmeta / dokumenata u rešenju **DocCentral**.

Koristi se kao glavna poslovna lista za zavođenje, praćenje statusa, povezivanje dokumenata, dodelu prava pristupa i prikaz predmeta u Canvas Power Apps aplikaciji.

## 2. Uloga u rešenju

**Poznato:**

- Lista je deo SharePoint Online data modela.
- Koristi se u procesu zavođenja dokumenata.
- Dokumenti mogu imati statuse kao što su **Zavedeno** i **Arhivirano**.
- Polje **DelovodniBroj** / **Title** se koristi kao poslovni identifikator dokumenta.
- Dokumenti se povezuju sa bibliotekama dokumenata preko vrednosti delovodnog broja.
- Nakon završetka procesa poslednja PDF verzija dokumenta se arhivira u SharePoint.
- Korisnici imaju read-only pristup SharePoint listama, dok upis treba da ide preko Power Automate flow-ova.

**Nepoznato:**

- Kompletna konfiguracija svih kolona nije potvrđena bez punog SharePoint REST exporta.
- Nisu potvrđeni svi interni nazivi kolona.
- Nisu potvrđeni svi indeksirani fieldovi.
- Nisu potvrđena sva podešavanja verzionisanja, validacije i unikatnosti.

## 3. Ključne kolone

| Display name | Interni naziv | Tip | Poznato / Nepoznato | Opis |
|---|---|---|---|---|
| Title | Title | Single line of text | Poznato | Često se koristi kao delovodni broj ili poslovni identifikator. |
| DelovodniBroj | NEPOZNATO | Single line of text | Delimično poznato | Koristi se za povezivanje stavke sa dokumentima u bibliotekama. |
| TipDokumenta | NEPOZNATO | Choice | Poznato | Tip dokumenta, primer: Fakture. |
| Stanje | NEPOZNATO | Choice/Text | Poznato | Status procesa, primer: Zavedeno, Arhivirano. |
| EDokument | NEPOZNATO | Yes/No | Poznato | Obeležava da li je dokument elektronski. |
| Partner | PartnerId | Lookup | Delimično poznato | Veza ka listi Partneri. |
| DatumRacuna | DatumRacuna | Date and Time | Poznato | Datum računa kada je predmet faktura. |
| DatumPrometa | DatumPrometa | Date and Time | Poznato | Datum prometa. |
| VrstaFakture | VrstaFakture | Choice | Poznato | Kategorija fakture. |
| Vrsta | Vrsta | Choice | Poznato | Primeri vrednosti: SEF, Fiskal, Običan. |
| OrginalKopija | OrginalKopija | Choice | Poznato | Primeri vrednosti: Mail, Orginal, Kopija. |
| Ukupno | Ukupno | Number | Poznato | Ukupan iznos. |
| Osnovica | Osnovica | Number | Poznato | Osnovica za obračun. |
| 20%PDV | OData__x0032_0PDV | Number | Poznato | PDV stopa 20%. |
| 10%PDV | OData__x0031_0PDV | Number | Poznato | PDV stopa 10%. |
| IskazanPDVKojiNijeOdbitni | IskazanPDVKojiNijeOdbitni | Number | Poznato | Iznos iskazanog PDV-a koji nije odbitni. |
| ProjectManager | ProjectManager | Person or Group | Poznato | Odgovorna osoba / project manager. |
| DodatniPrimaoci | DodatniPrimaoci | Person or Group | Poznato | Dodatni primaoci. |
| Odobrava | Odobrava | Person or Group | Poznato | Osoba koja odobrava. |
| Projekat | Projekat | Single line of text | Poznato | Naziv ili oznaka projekta. |
| Poreski status | NEPOZNATO | Choice/Text | Delimično poznato | Poreski status dokumenta. |
| Napomena | Napomena | Multiple lines of text | Poznato | Slobodan tekst / komentar. |
| InterniBroj | InterniBroj | Single line of text | Poznato | Interni broj dokumenta. |
| RunFlow | RunFlow | Single line of text | Poznato | Tehničko polje za pokretanje ili kontrolu flow logike. |
| NumID | NumID | Number | Poznato | Redni broj / pomoćni numerički ID. |

## 4. Poznate Choice vrednosti

### VrstaFakture

Poznate vrednosti:

- Izlazne fakture
- Izlazni avansni računi
- Ulazne fakture
- Ulazni avansni računi
- Ulazno knjižno odobrenje
- Ulazno knjižno zaduženje
- Izlazno knjižno odobrenje
- Izlazno knjižno zaduženje

### Vrsta

Poznate vrednosti:

- SEF
- Fiskal
- Običan

### OrginalKopija

Poznate vrednosti:

- Mail
- Orginal
- Kopija

## 5. Veze sa drugim listama i bibliotekama

| Povezani objekat | Tip veze | Opis |
|---|---|---|
| Partneri | Lookup | Stavka može biti povezana sa partnerom. |
| Dokumenta biblioteka | DelovodniBroj / metadata link | Dokumenti se filtriraju po delovodnom broju. |
| HG / druge biblioteke | DelovodniBroj / metadata link | Koristi se za prikaz i arhiviranje dokumenata. |
| App Config | Konfiguracija / šifarnici | Aplikacija koristi konfiguracione vrednosti i šifarnike. |
| Rezervisani brojevi | Poslovna kontrola numeracije | Potencijalno vezano za sprečavanje duplikata delovodnog broja. |

## 6. Poznata Power Apps logika

**Poznato:**

- Aplikacija čita podatke preko Power Automate flow-ova koji vraćaju JSON.
- Canvas app kreira lokalne kolekcije na osnovu JSON odgovora.
- Upis ne treba da ide direktno iz aplikacije kada korisnici imaju SharePoint read-only prava.
- Create/Edit/Delete operacije treba da idu preko Power Automate flow-ova.
- Forma treba da prikazuje samo polja koja su vidljiva za dati režim: New, Edit, View.
- Koristi se srpski lokalni format za brojeve i datume.

## 7. Poznata Power Automate logika

**Poznato:**

- Flow-ovi se koriste za čitanje SharePoint stavki i vraćanje JSON-a u Power Apps.
- Flow-ovi se koriste za upis i izmenu stavki pod servisnim nalogom.
- Postoje trigger condition obrasci vezani za polja kao što su Stanje, TipDokumenta i EDokument.
- Bitan enterprise zahtev je da više korisnika može istovremeno da zavodi dokumenta, ali da sistem nikada ne sme dodeliti isti delovodni broj za dva dokumenta.

## 8. Kritični enterprise zahtevi

### 8.1 Concurrency-safe delovodni broj

Obavezno implementirati:

- atomic ili optimistic locking mehanizam;
- retry logiku;
- unique constraint gde je moguće;
- audit log;
- error log;
- jasno korisničko obaveštenje ako broj nije mogao biti dodeljen.

### 8.2 Sigurnost

Preporučeni model:

- korisnici imaju read-only pristup SharePoint listama;
- Power Automate servisni nalog izvršava sve write operacije;
- Power Apps ne sme direktno da radi Patch za kritične poslovne upise ako korisnici nemaju write pravo;
- prava nad dokumentima se dodeljuju kroz grupe / flow logiku.

## 9. Poznate nepoznanice

- Tačan interni naziv svake kolone.
- Da li je Title enforced unique.
- Da li postoji posebno polje DelovodniBroj ili se koristi Title.
- Koje kolone su indexed.
- Koje kolone su required.
- Da li lista ima item-level permissions.
- Da li postoji content approval.
- Da li postoji versioning.
- Tačan REST schema export.
- Tačni Power Automate flow nazivi koji čitaju / pišu u ovu listu.

## 10. Preporuka za dalju dokumentaciju

Potrebno je dopuniti ovaj dokument iz SharePoint REST exporta sledećim informacijama:

- `Title`
- `InternalName`
- `TypeAsString`
- `Required`
- `ReadOnlyField`
- `Hidden`
- `Indexed`
- `EnforceUniqueValues`
- `Choices`
- `LookupList`
- `LookupField`
- `DefaultValue`
