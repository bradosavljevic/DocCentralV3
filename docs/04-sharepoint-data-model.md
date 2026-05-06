# 04 - SharePoint Data Model

## Svi predmeti

Glavna lista za metadata svih dokumenata.

Ključna polja:
- DelovodniBroj
- Dokument
- VrstaDokumenta
- DelovodnaKnjiga
- Stanje
- DatumZavodjenja
- ZaduzenaOsoba
- ImePartnera
- OrganizacionaJedinica
- KlasifikacionaOznaka
- DokumentIznos
- EDokument
- TipDokumenta
- LinkDoDokumenta
- Arhivirano
- Arhivirao
- TrenutniOdobravalacEmail
- CurrentApproverGroupId
- CurrentApprovalStep
- TotalApprovalSteps
- ApprovalStepsJson
- ApprovalHistoryJson
- InitiatorEmail
- TechnicalStatus

Pravila:
- `Svi predmeti` je izvor istine za dodeljen `DelovodniBroj`.
- `DelovodniBroj` mora biti unique i indexed.
- Partner se čuva kao tekstualni snapshot `ImePartnera`, ne kao lookup.
- Item može kratko postojati bez `DelovodniBroj` samo dok traje transaction.

## Dokumenta

Glavna biblioteka za fizičke dokumente i priloge.

Pravila:
- glavni dokument je obavezan
- prilog je opcionalan
- glavni dokument: `Attachment = false`
- prilog: `Attachment = true`
- glavni dokument i prilozi dele isti `DelovodniBroj`
- metadata mora biti sinhronizovan sa `Svi predmeti`

## EmailDocuments

Staging biblioteka za dokumente pristigle emailom. Pri obradi se fajl kopira u `Dokumenta`, a metadata se upisuje u `Svi predmeti`.

## AppConfig

Centralni JSON configuration store. Promene moraju ići preko Power Automate i moraju imati JSON/schema validaciju.

## Rezervisani brojevi

Rezervisani brojevi važe samo za aktivnu godinu, mogu se koristiti za bilo koji tip dokumenta i nakon uspešne upotrebe moraju se obrisati iz liste.

## Partneri

Posebna lista partnera. Brisanje partnera ne sme obrisati istorijsku vrednost `ImePartnera`.

## AuditLog

Log/history lista, retention 6 meseci, scheduled cleanup jednom nedeljno.

## NumberLocks / ZakljucavanjeBrojeva

Nova lista potrebna za atomarnu dodelu delovodnog broja.
