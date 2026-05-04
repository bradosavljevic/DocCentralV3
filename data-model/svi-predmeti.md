# SharePoint lista: Svi predmeti

## Svrha

Centralna lista za registrovane dokumente/predmete.

Dokumenti se gledaju direktno u SharePoint listama, ne kroz aplikacijski dashboard.

## Ključna polja

Napomena: tačna interna imena kolona treba uzeti iz XML/solution exporta. Često su interna imena jednaka display name-u bez razmaka.

Očekivana ključna polja:

- ID
- Title
- DelovodniBroj
- TipDokumenta
- Stanje
- DatumZavodjenja
- Godina
- Partner
- PartnerNaziv
- PartnerPIB
- Inicijator
- Odobrava
- DodatniPrimaoci
- OrganizacioneJedinice
- LinkDoDokumenta
- ArhivskiZnak
- DatumArhiviranja
- Created
- Modified
- Author
- Editor

## Statusi

Osnovni statusi:

- Zavedeno
- U odobravanju
- Odobreno
- Odbijeno
- Arhivirano

Kroz ProcesConfig mogu postojati dodatni statusi.

## Pravila

- `DelovodniBroj` mora biti unikatan.
- Dokument mora zadržati istoriju partnera čak i ako partner bude obrisan ili deaktiviran.
- Korisnici vide samo dokumente za koje imaju SharePoint permissions.
- Backend permissions su obavezni.
- UI filtriranje nije dovoljno.

## Permission model

Item/folder/file permissions se dodeljuju kroz break inheritance.

- service account: RW
- owners: RW
- members/viewers: Read
- relevantni korisnici/grupe: Read

## Napomena za Claude Code

Ne pretpostavljati da postoji ekran "Svi predmeti" u Canvas aplikaciji.

Svi predmeti je SharePoint lista, ne aplikacijski modul za pregled svih dokumenata.
