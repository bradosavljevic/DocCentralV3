# Skill: Power Apps Localization

## Cilj

Aplikacija mora podržati srpski i engleski jezik. Srpski je podrazumevani jezik.

## Pravila

- Ne hardkodovati tekstove direktno kroz aplikaciju ako je predviđena lokalizacija.
- Tekstove držati u konfiguraciji, kolekciji ili translation tabeli.
- Svi label tekstovi, poruke greške, statusi i button tekstovi treba da imaju translation key.
- Koristiti konzistentan format za datume i brojeve.

## Srpski format

- Datumi: `dd.MM.yyyy` ili poslovno definisani format.
- Decimalni separator: zarez.
- Hiljadarski separator: tačka.
- Power Apps formule treba da koriste eksplicitni locale kada se radi formatiranje/parsing.

## Preporuka za translation funkciju

Koristiti named formula ili helper logiku tipa:

```powerfx
nfTranslate(key: Text): Text
```

Funkcija treba da vrati prevod na osnovu trenutno izabranog jezika.

## Jezik

- Default: `sr-Latn-RS` ili poslovno definisana srpska latinica.
- Dodatni: `en-US` ili `en-GB`, zavisno od odluke.

Ako nije potvrđeno koji engleski locale se koristi, označiti kao `NEPOZNATO`.
