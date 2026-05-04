# Checklist: kvalitet aplikacije

## Canvas App

- [ ] Responsive design
- [ ] Clean App Checker
- [ ] Nema cross-screen control references
- [ ] Nema nested galleries osim ako je opravdano
- [ ] Nema live search query-ja na svaki karakter
- [ ] Delegabilne formule gde god je moguće
- [ ] `IfError` oko Patch i Flow poziva
- [ ] `gbl`, `loc`, `col` naming konvencije

## Power Automate

- [ ] Try/Catch/Finally scopes
- [ ] Standardizovan response prema Canvas aplikaciji
- [ ] Audit log u Catch granama
- [ ] CorrelationId u svim glavnim procesima
- [ ] Retry logika za delovodni broj
- [ ] Bez folder kreiranja za dokumente
- [ ] Unikatan naziv za svaki fajl

## SharePoint

- [ ] `DelovodniBroj` unikatan
- [ ] Item/file permissions se fizički primenjuju
- [ ] Prilozi povezani metapodacima
- [ ] Partner snapshot podaci očuvani
- [ ] AuditLog lista postoji
