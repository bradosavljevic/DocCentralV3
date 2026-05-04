# Test matrica

## 1. Unos novog dokumenta

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-001 | Korisnik zavodi dokument sa sledećim brojem | Dokument kreiran, broj dodeljen, brojač uvećan |
| T-002 | Korisnik zavodi dokument sa rezervisanim brojem | Dokument kreiran, rezervisani broj obrisan |
| T-003 | Nedostaje obavezno polje | Sistem prikazuje validacionu grešku |
| T-004 | Nedostaje prilog | Sistem prikazuje validacionu grešku |
| T-005 | Godina je zaključana | Zavođenje nije dozvoljeno |
| T-006 | Dva korisnika istovremeno kliknu Zavedi | Dokumenti dobijaju različite delovodne brojeve |

## 2. Rezervisani brojevi

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-010 | Korisnik kreira rezervisani broj | Broj kreiran |
| T-011 | Korisnik menja rezervisani broj | Broj izmenjen |
| T-012 | Korisnik pokušava ručno da obriše rezervisani broj | Brisanje nije dozvoljeno |
| T-013 | Rezervisani broj se koristi kod zavođenja | Broj se briše nakon uspešnog zavođenja |
| T-014 | Kreiranje dokumenta ne uspe | Rezervisani broj ostaje u listi |

## 3. Odobravanje

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-020 | Dokument ide jednom korisniku | Korisnik vidi dokument za odobravanje |
| T-021 | Sekvencijalno odobravanje | Sledeći korisnik dobija tek nakon prethodnog |
| T-022 | Odobravanje na grupu | Prvi koji odobri završava korak |
| T-023 | Odbijanje | Status Odbijeno, dokument se vraća inicijatoru |
| T-024 | Inicijator ponovo pokreće approval | Kreira se novi approval proces |

## 4. Podsetnici

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-030 | Korisnik kreira podsetnik za sebe | Podsetnik kreiran |
| T-031 | Korisnik kreira podsetnik za više korisnika | Svi primaoci dobijaju email |
| T-032 | Dan podsetnika | Email se šalje u 08:00 |
| T-033 | Podsetnik već poslat | Ne šalje se ponovo |
| T-034 | Korisnik briše podsetnik | Email se ne šalje |

## 5. Arhiviranje

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-040 | Arhiviranje dokumenta u statusu Zavedeno | Status postaje Arhivirano |
| T-041 | Arhiviranje dokumenta koji nije Zavedeno | Sistem odbija operaciju |
| T-042 | Dodela arhivskog znaka | Arhivski znak upisan |
| T-043 | Generisanje Arhivske knjige PDF | PDF kreiran |

## 6. Zaključenje godine

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-050 | Svi dokumenti su Arhivirano i nema rezervisanih brojeva | Godina zaključana |
| T-051 | Postoji dokument koji nije Arhivirano | Zaključenje odbijeno |
| T-052 | Postoji rezervisani broj | Zaključenje odbijeno |
| T-053 | Nakon zaključenja pokušaj zavođenja u staroj godini | Nije dozvoljeno |
| T-054 | Pokušaj otključavanja godine | Nije dozvoljeno |

## 7. Partneri

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-060 | Kreiranje partnera | Partner kreiran |
| T-061 | Izmena partnera | Partner izmenjen |
| T-062 | Brisanje/deaktivacija partnera | Partner nedostupan za nove dokumente |
| T-063 | Istorijski dokument sa obrisanim partnerom | Istorija ostaje vidljiva |

## 8. Security

| ID | Scenario | Očekivani rezultat |
|---|---|---|
| T-070 | Korisnik bez prava otvara dokument | Nema pristup |
| T-071 | Korisnik sa pravom otvara dokument | Ima Read pristup |
| T-072 | Korisnik pokušava direktan edit SharePoint item-a | Nema Write pristup |
| T-073 | Flow kreira dokument | Uspelo preko service account-a |
