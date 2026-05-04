# Skill: Delovodni Broj Concurrency

## Cilj

Obezbediti da više korisnika može istovremeno da zavodi dokumente, ali da dva dokumenta nikada ne dobiju isti `DelovodniBroj`.

## Kritično pravilo

Claude Code nikada ne sme projektovati dodelu delovodnog broja samo u Canvas aplikaciji, lokalnoj kolekciji ili formulom tipa `Max(...) + 1`.

## Poznate činjenice

- `DelovodniBroj` mora biti unikatan.
- `App Config` sadrži konfiguraciju `Delovodne knjige` sa sledećim brojem.
- Postoji lista `Rezervisani Brojevi`.
- ETag se trenutno ne koristi, ali ga treba razmotriti.
- Retry logika trenutno nije potvrđena, ali treba da postoji u novoj verziji.
- Audit neuspelih pokušaja trenutno nije potvrđen, ali treba da postoji.
- Svake godine delovodnik se zaključava i kreira se arhivska knjiga za zavedene/arhivirane dokumente.

## Ispravan obrazac

```text
Canvas App
  -> Power Automate: Reserve/Register Number
    -> read App Config / Delovodne knjige
    -> attempt atomic reservation
    -> write Rezervisani Brojevi
    -> create/update Svi predmeti
    -> update next number
    -> write audit log
    -> return confirmed number
```

## Obavezni elementi

- Backend rezervacija broja.
- Unique constraint nad `DelovodniBroj` ili ekvivalentna SharePoint kontrola.
- Retry na konflikt.
- Audit svakog pokušaja.
- CorrelationId.
- Jasna greška ako rezervacija ne uspe.
- Nikada ne prikazati broj kao finalan dok backend ne potvrdi uspeh.

## Preporučena strategija

1. Flow prima zahtev za zavođenje.
2. Flow čita aktivnu delovodnu knjigu iz `App Config`.
3. Flow određuje kandidat broj.
4. Flow pokušava upis u `Rezervisani Brojevi` sa unique vrednošću.
5. Ako upis uspe, broj je rezervisan.
6. Ako upis ne uspe zbog konflikta, flow povećava kandidat broj i pokušava ponovo.
7. Nakon uspešne rezervacije, flow upisuje/ažurira `Svi predmeti`.
8. Flow ažurira sledeći broj u `App Config`.
9. Flow vraća potvrđen `DelovodniBroj` Canvas aplikaciji.

## ETag / optimistic locking

Preporuka je razmotriti ETag nad konfiguracionim zapisom delovodne knjige.

Ako se koristi ETag:

- pročitati item i ETag;
- update izvršiti samo ako se ETag nije promenio;
- ako se promenio, ponoviti čitanje i retry;
- logovati konflikt.

## Zabranjeno

- Rezervisati broj samo prikazom u aplikaciji.
- Koristiti samo `Max(DelovodniBroj)+1` bez unique constraint-a.
- Ignorisati grešku SharePoint unique constraint-a.
- Kreirati predmet bez potvrđenog broja ako je broj obavezan za status `Zavedeno`.
