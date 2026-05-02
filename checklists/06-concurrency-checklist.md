# Concurrency Checklist

## Kritični scenario

Više korisnika istovremeno zavodi dokumenta.

Sistem nikada ne sme dozvoliti dupliran delovodni broj.

## Provera trenutnog stanja

- [ ] Da li se delovodni broj generiše u Canvas app-u?
- [ ] Da li se delovodni broj generiše u Power Automate-u?
- [ ] Da li postoji centralni counter?
- [ ] Da li Title ili DelovodniBroj ima unique constraint?
- [ ] Da li postoji retry logika?
- [ ] Da li postoji lock mehanizam?
- [ ] Da li postoji audit log?
- [ ] Da li postoji error log?
- [ ] Da li korisnik dobija jasnu poruku u slučaju konflikta?

## Preporučeni enterprise model

- [ ] Centralizovana lista za countere
- [ ] Atomic update ili optimistic locking
- [ ] Unique constraint na delovodni broj
- [ ] Retry sa ograničenim brojem pokušaja
- [ ] Error log za neuspešne pokušaje
- [ ] Audit log za uspešno generisanje
- [ ] Power Apps ne generiše finalni broj samostalno
- [ ] Power Automate vraća finalni broj aplikaciji

## Test scenariji

- [ ] Dva korisnika istovremeno klikću submit
- [ ] Pet korisnika istovremeno klikću submit
- [ ] Flow dobija conflict grešku
- [ ] Retry uspeva
- [ ] Retry ne uspeva
- [ ] Korisnik dobija jasnu poruku
- [ ] Nijedan duplikat nije kreiran
