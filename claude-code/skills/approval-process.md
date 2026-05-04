# Skill: Approval Process

## Cilj

Definisati obrazac za odobravanje dokumenata, uz jasno označavanje nepoznanica.

## Nepoznato / mora se potvrditi

- Ko sve učestvuje u odobravanju.
- Da li je odobravanje sekvencijalno ili paralelno.
- Da li se koristi Power Automate Approvals konektor.
- Gde se čuvaju odgovori odobravača.
- Šta se dešava kod odbijanja.
- Da li postoji vraćanje na doradu.
- Da li postoji bulk approval.
- Da li postoje SLA rokovi.
- Da li se šalju email/Teams notifikacije.

## Preporučeni model

- Approval pravila definisati u konfiguraciji ako treba da budu fleksibilna.
- Svaka approval odluka mora biti auditovana.
- Odgovori odobravača moraju biti povezani sa predmetom/dokumentom.
- Kod odbijanja mora postojati razlog.
- UI mora jasno prikazati trenutni approval status.

## Flow pravila

- Approval flow mora imati error handling.
- Ne sme zaključati dokument bez potvrđenog finalnog statusa.
- Mora vratiti jasnu informaciju aplikaciji ili upisati status za kasnije čitanje.

## Bulk approval

Ako se implementira:

- korisnik sme odobriti samo dokumente za koje ima pravo;
- rezultat mora prikazati uspešne i neuspešne stavke;
- svaka stavka mora imati audit log;
- greška jedne stavke ne sme sakriti rezultat ostalih.
