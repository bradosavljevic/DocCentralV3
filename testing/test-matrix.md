# Test matrica

## Unos dokumenta

| Scenario | Očekivani rezultat |
|---|---|
| Korisnik zavodi dokument sa novim brojem | Dokument dobija unikatan delovodni broj |
| Korisnik zavodi dokument sa rezervisanim brojem | Rezervisani broj se koristi i uklanja iz liste |
| Dva korisnika istovremeno zavode dokument | Svaki dokument dobija različit broj |
| Korisnik doda prilog | Prilog se kreira u root biblioteke i povezuje metapodacima |
| Fajl ima isto originalno ime kao postojeći | Ne dolazi do overwrite-a |
| Nedostaje obavezno polje | Dokument se ne zavodi, korisnik dobija poruku |

## Odobravanje

| Scenario | Očekivani rezultat |
|---|---|
| Dokument ide jednom korisniku | Korisnik vidi dokument za odobrenje |
| Dokument ide sekvencijalno | Sledeći odobravač dobija dokument tek nakon prethodnog |
| Approval ide grupi | Prvi koji odobri završava korak |
| Dokument je odbijen | Status postaje `Odbijeno` i vraća se inicijatoru |
| Inicijator ponovo šalje dokument | Proces odobravanja se ponovo pokreće |

## Arhiviranje

| Scenario | Očekivani rezultat |
|---|---|
| Dokument je `Zavedeno` | Može se arhivirati |
| Dokument nije `Zavedeno` | Ne može direktno u arhivu |
| Arhiviranje uspešno | Status postaje `Arhivirano` |
| Arhiviranje neuspešno | Greška se loguje |

## Zaključenje godine

| Scenario | Očekivani rezultat |
|---|---|
| Svi dokumenti su `Arhivirano`, nema rezervisanih brojeva | Godina se zaključuje |
| Postoji dokument koji nije `Arhivirano` | Zaključenje nije dozvoljeno |
| Postoji rezervisani broj | Zaključenje nije dozvoljeno |
| Godina je zaključana | Nikada se ne može otključati |
| Zaključenje uspešno | Kreira se nova delovodna knjiga i menja aktivna godina |

## Partneri

| Scenario | Očekivani rezultat |
|---|---|
| Kreiranje partnera | Partner je dostupan za izbor |
| Izmena partnera | Novi podaci su dostupni za buduće dokumente |
| Brisanje partnera | Partner više nije dostupan za izbor |
| Pregled starog dokumenta | Istorijski podaci partnera ostaju vidljivi |

## Podsetnici

| Scenario | Očekivani rezultat |
|---|---|
| Korisnik kreira podsetnik za sebe | Email se šalje na datum podsetnika |
| Korisnik kreira podsetnik za više korisnika | Svi primaoci dobijaju email |
| Podsetnik se menja | Koristi se nova vrednost |
| Podsetnik se briše | Email se ne šalje |
| Dan podsetnika je stigao | Email se šalje u 08:00 ili prema konfiguraciji |

## Audit

| Scenario | Očekivani rezultat |
|---|---|
| Neuspešno kreiranje dokumenta | Upis u AuditLog |
| Uspešno generisan broj | Upis u AuditLog |
| Neuspešno generisanje broja | Upis u AuditLog |
| Korišćen rezervisani broj | Upis u AuditLog |
| Flow greška | Upis u AuditLog |
