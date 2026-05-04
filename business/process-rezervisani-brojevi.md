# Proces: Rezervisani delovodni brojevi

## Cilj

Omogućiti unapred rezervisanje delovodnog broja za kasnije zavođenje dokumenta.

## Ko može da kreira rezervisani broj

Rezervisane brojeve mogu kreirati korisnici koji zavode dokumente.

## Pravila

- Rezervisani broj važi samo za jednu godinu.
- Rezervisani broj može se koristiti za bilo koji tip dokumenta.
- Svi korisnici koji zavode dokumente vide sve rezervisane brojeve.
- Rezervisani broj može se izmeniti.
- Nakon rezervacije broj ne može biti ručno obrisan.
- Rezervisani broj se briše automatski tek kada se iskoristi za zavođenje.
- Lista rezervisanih brojeva mora biti prazna pre zaključenja godine.

## Korišćenje

Kada korisnik kod zavođenja izabere rezervisani broj:

1. bira rezervisani broj
2. bira datum zavođenja
3. sistem validira da je broj dostupan
4. sistem zavodi dokument sa tim brojem
5. sistem briše rezervisani broj iz liste

## Logging

Obavezno logovati:

- korišćenje rezervisanog broja
- neuspešno korišćenje rezervisanog broja
- neuspešno brisanje iskorišćenog rezervisanog broja
