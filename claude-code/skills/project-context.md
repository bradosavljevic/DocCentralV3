# Skill: Project Context - DocCentral / e-pisarnica

## Svrha

Claude Code mora razumeti da je DocCentral poslovno rešenje za elektronsku pisarnicu, zavođenje, upravljanje, pregled, odobravanje i arhiviranje dokumenata.

## Činjenice

- Rešenje se zasniva na Power Apps Canvas aplikaciji, Power Automate flow-ovima i SharePoint Online listama/bibliotekama.
- SharePoint je glavni data/document storage sloj.
- Korisnici imaju Read Only pristup SharePoint-u.
- Upisi, izmene, brisanja i operacije nad dokumentima moraju ići preko Power Automate-a.
- Power Automate radi pod servisnim nalogom sa RW pravima.
- Pristup dokumentima zavisi od organizacionih jedinica i Entra ID / Microsoft 365 grupa.
- Security se mora sprovoditi i na SharePoint backend-u, ne samo u Canvas aplikaciji.
- Delovodni broj je kritična poslovna funkcionalnost.
- Više korisnika mora moći istovremeno da zavodi dokumenta, ali sistem nikada ne sme dozvoliti dupli delovodni broj.

## Važni domeni

- `Svi predmeti` - centralna evidencija predmeta/dokumenata.
- `Partneri` - evidencija poslovnih partnera.
- `App Config` - šifarnici i konfiguracioni JSON zapisi.
- `Rezervisani Brojevi` - rezervacije delovodnih brojeva.
- Dokument biblioteke - skladištenje fajlova i metapodataka.

## Pravila

Claude Code mora:

1. koristiti konkretne nazive lista, kolona, flow-ova i konfiguracija kada postoje u dokumentaciji;
2. označiti nepoznate podatke kao `NEPOZNATO`;
3. ne sme uvoditi premium konektore bez eksplicitne odluke;
4. ne sme projektovati direktne SharePoint write operacije iz Canvas aplikacije za krajnje korisnike;
5. mora tretirati concurrency kao obavezan enterprise zahtev;
6. mora dokumentovati svaku pretpostavku.

## Ignoriše se za sada

- SAP / eksterni import, osim ako korisnik kasnije eksplicitno traži njegovu razradu.
