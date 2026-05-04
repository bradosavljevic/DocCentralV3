# Skill: SharePoint Document Libraries

## Cilj

Biblioteke dokumenata moraju podržati sigurno čuvanje, pretragu, povezivanje sa predmetima i arhiviranje.

## Pravila

- Svaki dokument mora biti povezan sa poslovnim predmetom ili delovodnim brojem kada je to primenljivo.
- Metapodaci moraju biti upisani kroz Power Automate kada korisnik nema write prava.
- Linkovi ka dokumentima moraju biti formirani bez invalid URL vrednosti.
- Dokumenti moraju imati odgovarajuća SharePoint prava.

## Preporučeni metapodaci

- `DelovodniBroj`
- `PredmetId`
- `TipDokumenta`
- `Status`
- `Partner`
- `OrganizacionaJedinica`
- `Godina`
- `DatumDokumenta`
- `DatumZavođenja`
- `Arhivirano`

Koristiti samo ako su potvrđeni ili projektovani kao nova verzija. Nepoznato označiti kao `NEPOZNATO`.

## Linkovi

Kod URL polja:

- proveriti da li se upisuje objekat ili string, zavisno od SharePoint kolone;
- enkodovati specijalne karaktere;
- ne upisivati prazan ili invalid URL;
- koristiti server-relative URL kada je pogodnije.

## Verzije

Ako je verzionisanje uključeno, dokumentovati:

- major/minor verzije;
- retention;
- ko može da menja dokument;
- šta se dešava nakon arhiviranja.
