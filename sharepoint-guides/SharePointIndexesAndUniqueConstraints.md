# SharePoint Indexes And Unique Constraints

## Required

| List | Column | Required setting |
|---|---|---|
| Svi predmeti | DelovodniBroj | Indexed + Unique |
| Rezervisani brojevi | RezervisaniBroj | Indexed + Unique |
| NumberLocks | LockKey | Indexed + Unique |

## Recommended indexes

| List | Column |
|---|---|
| Svi predmeti | Stanje |
| Svi predmeti | DatumZavodjenja |
| Svi predmeti | TipDokumenta |
| Svi predmeti | OrganizacionaJedinica |
| Partneri | PIB |
| Dokumenta | DelovodniBroj |
| Dokumenta | Attachment |
| EmailDocuments | Processed/Status if implemented |
| AuditLog | Created |
