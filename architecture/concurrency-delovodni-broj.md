# Concurrency-safe delovodni broj

## Kritičan zahtev

Sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.

Ovo je obavezan enterprise zahtev.

## Poznato trenutno stanje

- Sledeći delovodni broj se čuva u App Config / Delovodne knjige.
- Postoji lista Rezervisani Brojevi.
- DelovodniBroj je unikatan.
- ETag se trenutno ne koristi.
- Retry logika trenutno ne postoji.
- Audit neuspelih pokušaja trenutno ne postoji, ali treba ga dodati.

## Cilj nove verzije

Generisanje broja mora biti atomic ili optimistic-locking safe.

## Preporučeni model

1. Power Apps poziva Power Automate flow `RegisterDocument`.
2. Flow dobija zahtev za zavođenje.
3. Flow čita aktivnu delovodnu knjigu.
4. Flow pokušava da rezerviše/generiše broj kroz kontrolisani update.
5. Flow koristi unique constraint na `DelovodniBroj` kao poslednju liniju zaštite.
6. Ako dođe do konflikta, flow radi retry sa novim brojem.
7. Ako retry ne uspe, flow vraća kontrolisanu grešku korisniku.
8. Svaki pokušaj se loguje.

## Preporučene zaštite

- Unique constraint na `DelovodniBroj`.
- Atomic update brojača.
- ETag / If-Match gde je moguće.
- Retry policy.
- Log lista za neuspešne i uspešne pokušaje.
- Jedan centralni flow za generisanje broja.
- Ne generisati broj direktno u Canvas aplikaciji.

## Minimalni flow koraci

1. Validate request.
2. Check active year.
3. Check year is not locked.
4. Determine number mode:
   - reserved number
   - next number
5. If reserved:
   - validate reserved number exists
   - validate year
   - create document
   - delete reserved number
6. If next number:
   - read counter
   - generate delovodni broj
   - create document
   - increment counter
7. Assign permissions.
8. Return success/failure to Power Apps.
9. Write audit log.

## Logovati

- pokušaj generisanja delovodnog broja
- uspešno generisan delovodni broj
- neuspešno generisanje broja
- retry attempt
- reserved number usage
- unique constraint conflict
