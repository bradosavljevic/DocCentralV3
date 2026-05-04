# Process: Approval

## Osnovni model

Dokument ide na odobrenje jednom korisniku u jednom koraku.

Ako postoji više korisnika, approval ide sekvencijalno.

Postoji mogućnost da korak ide na grupu. Tada više korisnika dobija obaveštenje, ali prvi korisnik koji odobri završava taj approval korak.

## Statusi

Approval menja status dokumenta preko polja `Stanje`.

Osnovni statusi:

- U odobravanju
- Odobreno
- Odbijeno

## Odbijanje

Ako korisnik odbije dokument:

- dokument dobija status `Odbijeno`
- vraća se inicijatoru
- inicijator menja dokument ili metadata
- inicijator može ponovo pokrenuti approval proces

## Vidljivost

U aplikaciji korisnik vidi dokumente koje nije odobrio.

U SharePoint listi korisnik vidi dokumente koji su za njega ili dokumente u kojima su on ili njegova grupa učestvovali u procesu odobravanja.

## Implementaciona napomena

Detaljan izbor između Power Automate Approvals konektora i interne SharePoint liste odobrenja treba proveriti iz solution-a ili definisati u novoj verziji.
