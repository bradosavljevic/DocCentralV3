# Skill: Power Automate Child Flow Patterns

## Cilj

Smanjiti dupliranje logike i standardizovati backend operacije kroz child flow-ove.

## Kandidati za child flow

- validacija korisnika i grupa;
- logovanje;
- error response builder;
- dodela SharePoint prava;
- rezervacija delovodnog broja;
- čitanje App Config vrednosti;
- parsiranje JSON konfiguracije;
- ažuriranje partnera;
- slanje notifikacija.

## Pravila

- Child flow mora imati jasno definisan input i output.
- Ne sme zavisiti od nevidljivog globalnog stanja.
- Mora vraćati strukturisan rezultat.
- Mora imati error handling.

## Naming convention

Preporuka:

```text
CF - <Domain> - <Action>
```

Primeri:

- `CF - Audit - Write Log`
- `CF - Numbering - Reserve Number`
- `CF - Security - Apply Item Permissions`
- `CF - Config - Get Active Key`
