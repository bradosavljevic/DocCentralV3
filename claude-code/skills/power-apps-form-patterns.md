# Skill: Power Apps Form Patterns

## Cilj

Definisati standard za unos, pregled, izmenu, zavođenje i arhiviranje dokumenata.

## Režimi forme

Svaki poslovni entitet treba imati jasne režime:

- `New` - unos novog zapisa
- `View` - pregled bez izmene
- `Edit` - izmena postojećeg zapisa
- `Archive` - arhiviranje / zaključavanje
- `Register` - zavođenje i dodela delovodnog broja

## Pravila za vidljivost polja

- Samo polja koja su poslovno potrebna treba prikazati korisniku.
- Vidljivost polja može zavisiti od tipa dokumenta, statusa, role, organizacione jedinice ili konfiguracije iz `App Config`.
- Ako pravilo nije poznato, označiti ga kao `NEPOZNATO`.

## Validacije

Validacije treba izvršiti pre poziva Power Automate flow-a:

- obavezna polja;
- format datuma;
- poslovna pravila za status;
- izbor partnera;
- postojanje dokumenta/priloga ako je potrebno;
- role/group dozvole za akciju.

Validacije u Canvas aplikaciji nisu zamena za backend validacije u Power Automate-u.

## Submit obrazac

Standardni obrazac:

1. Set loading state.
2. Uradi lokalne validacije.
3. Pripremi payload kao JSON.
4. Pozovi Power Automate flow.
5. Obradi success/error response.
6. Osveži kolekcije/podatke.
7. Prikaži korisniku jasnu poruku.

## Zabranjeno

- Direktan `Patch()` za poslovni upis ako korisnici imaju Read Only prava.
- Kritična validacija samo na UI nivou.
- Nastavak procesa ako backend nije potvrdio uspeh.
