# Skill: Power Automate SharePoint Write Pattern

## Cilj

Definisati standard za bezbedan upis u SharePoint kroz Power Automate.

## Obavezni tok

1. Primi payload iz Canvas aplikacije.
2. Generiši `correlationId`.
3. Validiraj obavezna polja.
4. Proveri poslovnu dozvolu korisnika.
5. Proveri da li item već postoji ako je update.
6. Izvrši Create/Update/Delete nad SharePoint-om servisnim nalogom.
7. Upisati audit log.
8. Vratiti strukturisan odgovor.

## Create

- Nikada ne oslanjati se samo na Canvas generisan ID.
- Za poslovne jedinstvene vrednosti koristiti SharePoint unique constraint, rezervaciju ili backend proveru.
- Za delovodni broj koristiti poseban concurrency-safe flow.

## Update

- Pre update-a učitati postojeći item.
- Proveriti status itema.
- Proveriti da li je item zaključan/arhiviran.
- Ako se koristi ETag, uporediti verziju.

## Delete

- Preferirati soft-delete/status, osim ako je poslovno dozvoljeno fizičko brisanje.
- Brisanje mora biti auditovano.

## Permissions

Nakon kreiranja ili izmene dokumenta, ako je potrebno:

- break inheritance;
- dodeliti RW servisnom nalogu i ownerima;
- dodeliti Read members/viewers grupama;
- ukloniti neželjene inherited permissione.
