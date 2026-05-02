# SharePoint lista: Rezervisani brojevi

## 1. Svrha liste

Lista **Rezervisani brojevi** služi ili može služiti za kontrolu, rezervaciju i audit delovodnih brojeva u DocCentral rešenju.

Ova lista je posebno važna zbog enterprise zahteva da više korisnika može istovremeno da zavodi dokumente, ali da sistem nikada ne sme dodeliti isti delovodni broj za dva dokumenta.

## 2. Status znanja

**Poznato:**

- U prethodnoj strukturi dokumentacije postojao je fajl `data-model/rezervisani-brojevi.md`.
- Kritično pravilo projekta je sprečavanje duplog delovodnog broja.
- Potrebna je concurrency-safe logika.

**Nepoznato:**

- Da li lista već postoji u SharePoint-u.
- Tačne kolone liste.
- Da li se trenutno koristi u produkcionoj logici.
- Da li Title ima enforced unique.

## 3. Predložena uloga u procesu numeracije

Lista treba da služi za atomic rezervaciju broja pre kreiranja finalnog predmeta.

Predloženi proces:

1. Korisnik pokrene zavođenje dokumenta.
2. Power Apps poziva Power Automate flow.
3. Flow izračunava predlog sledećeg broja.
4. Flow pokušava da kreira stavku u listi **Rezervisani brojevi** sa unique vrednošću broja.
5. Ako kreiranje uspe, broj je rezervisan.
6. Flow kreira stavku u **Svi predmeti** sa tim brojem.
7. Ako kreiranje predmeta uspe, rezervacija dobija status `Used`.
8. Ako ne uspe, rezervacija dobija status `Failed` ili `Expired`.

## 4. Predložene kolone

| Kolona | Tip | Svrha |
|---|---|---|
| Title | Single line of text | Delovodni broj. Preporuka: Enforce unique values = Yes. |
| DelovodniBroj | Single line of text | Ako se Title ne koristi kao broj. |
| Godina | Number | Godina numeracije. |
| Prefix | Single line of text | Prefiks, npr. Os.Del.Br. |
| RedniBroj | Number | Numerički deo broja. |
| Status | Choice | Reserved, Used, Failed, Expired, Cancelled. |
| ReservedBy | Person or Group | Korisnik koji je pokrenuo rezervaciju. |
| ReservedAt | Date and Time | Vreme rezervacije. |
| UsedAt | Date and Time | Vreme finalnog upisa. |
| PredmetId | Number / Lookup | ID kreirane stavke u Svi predmeti. |
| FlowRunId | Single line of text | ID Power Automate run-a. |
| ErrorMessage | Multiple lines of text | Greška ako upis nije uspeo. |
| RetryCount | Number | Broj pokušaja. |

## 5. Concurrency-safe princip

Najvažnije pravilo:

> Ne sme se oslanjati samo na lokalnu kolekciju u Power Apps ili na `Max(ID)+1` bez server-side zaštite.

Obavezno koristiti:

- server-side rezervaciju;
- unique constraint;
- retry logiku;
- kontrolu grešaka;
- audit zapis.

## 6. Retry logika

Predlog:

1. Izračunaj kandidat broj.
2. Pokušaj insert u Rezervisani brojevi.
3. Ako SharePoint vrati unique violation:
   - povećaj broj za 1;
   - pokušaj ponovo.
4. Ograničiti broj pokušaja, npr. 3 do 5.
5. Ako svi pokušaji padnu:
   - vratiti kontrolisanu grešku u Power Apps;
   - upisati error log.

## 7. Statusi

| Status | Opis |
|---|---|
| Reserved | Broj je rezervisan, ali predmet još nije kreiran. |
| Used | Broj je uspešno iskorišćen. |
| Failed | Rezervacija je uspela, ali finalni upis nije. |
| Expired | Rezervacija je istekla bez upotrebe. |
| Cancelled | Rezervacija je ručno ili sistemski poništena. |

## 8. Rizici ako ova lista ne postoji

- Dva korisnika mogu dobiti isti broj u isto vreme.
- Lokalna Power Apps kolekcija može biti zastarela.
- SharePoint delegacija može vratiti nepotpun rezultat.
- `Max()+1` obrazac nije bezbedan pri paralelnom radu.
- Naknadno rešavanje duplikata je poslovno i tehnički skupo.

## 9. Poznate nepoznanice

- Da li je lista već deo solution-a.
- Da li Title ima unique constraint.
- Koji flow koristi ovu listu.
- Da li postoji audit log.
- Da li postoji retry implementacija.
- Da li postoje expired rezervacije i cleanup job.
