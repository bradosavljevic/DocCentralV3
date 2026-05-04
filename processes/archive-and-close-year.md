# Proces: Arhiviranje i zaključenje godine

## Arhiviranje

Direktan prelaz iz `Zavedeno` u `Arhivirano` je dozvoljen.

To je jedini direktni put arhiviranja.

Aplikacija mora imati poseban ekran za arhiviranje.

Na ekranu za arhiviranje korisnik vidi dokumente:

- za tekuću godinu
- u statusu `Zavedeno`

Korisnik ih arhivira sa odgovarajućim arhivskim znacima.

## Flow

```text
CF_DocCentralV3_ArchiveDocument
```

## Zaključenje godine

Zaključenje godine se izvršava kroz:

```text
CF_DocCentralV3_CloseRegistryYear
```

## Uslovi za zaključenje

Svi uslovi moraju biti ispunjeni:

- svi dokumenti iz tekuće godine su u statusu `Arhivirano`
- ne postoji dokument iz tekuće godine u bilo kom drugom statusu
- lista rezervisanih brojeva za tu godinu je prazna
- kreira se nova delovodna knjiga za sledeću godinu
- aktivna godina se menja u App Config

## Pravilo zaključane godine

Zaključana godina se nikada ne može ponovo otključati.

## PDF arhivska knjiga

PDF generisanje ulazi u prvu verziju samo za Arhivsku knjigu.

Flow:

```text
CF_DocCentralV3_GenerateArchiveBookPdf
```

## Audit log

Obavezno logovati:

- arhiviranje
- neuspešno arhiviranje
- zaključenje godine
- neuspešno zaključenje godine
- generisanje PDF arhivske knjige
- greške flow-a
