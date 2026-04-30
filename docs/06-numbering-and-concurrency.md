# 06 — Numbering and concurrency

## 1. Svrha dokumenta

Ovaj dokument opisuje kritični deo DocCentral v6.0 rešenja: generisanje, rezervaciju i finalnu dodelu delovodnog broja.

Ovo je najvažniji enterprise zahtev sistema:

```text
Više korisnika mora moći istovremeno da zavodi dokumenta,
ali sistem nikada ne sme dozvoliti da dva dokumenta dobiju isti delovodni broj.
```

Dokument definiše:

- postojeće potvrđene elemente
- rizike trenutnog modela
- zašto Canvas app ne sme sama da generiše broj
- preporučeni backend model
- optimistic locking pristup
- retry logiku
- unique constraint
- audit log
- error handling
- predlog ciljane arhitekture za novu verziju

---

## 2. Status analize

Status:

```text
DELIMIČNO POTVRĐENO
```

Potvrđeno:

- postoji lista `RezervisaniBrojevi`
- lista ima polje `RezervisaniBroj`
- lista ima polje `DatumRezervacije`
- postoji biblioteka `Shared Documents`
- biblioteka `Shared Documents` ima polje `DelovodniBroj`
- `DelovodniBroj` je tekstualno polje
- nije potvrđeno da `DelovodniBroj` ima unique constraint
- nije dostavljena flow logika za generisanje broja

Nije potvrđeno:

- kako se broj trenutno generiše
- da li broj generiše Canvas app ili Power Automate
- da li postoji locking
- da li postoji retry
- da li postoji audit
- da li postoji centralni counter
- da li postoje različite delovodne knjige
- da li se broj razlikuje po godini, tipu dokumenta ili organizacionoj jedinici

---

## 3. Potvrđeni SharePoint elementi

## 3.1 Lista `RezervisaniBrojevi`

Scope:

```text
/sites/DocumentCentralv6.0/Lists/RezervisaniBrojevi
```

Potvrđena polja:

| Internal name | Display name | Tip | Status |
|---|---|---|---|
| `Title` | Title | Single line of text | Potvrđeno |
| `RezervisaniBroj` | Rezervisani broj | Number | Potvrđeno |
| `DatumRezervacije` | DatumRezervacije | Date and Time | Potvrđeno |
| `ID` | ID | Counter | Potvrđeno |
| `Created` | Created | Date and Time | Potvrđeno |
| `Modified` | Modified | Date and Time | Potvrđeno |
| `Author` | Created By | Person or Group | Potvrđeno |
| `Editor` | Modified By | Person or Group | Potvrđeno |

Zaključak:

```text
Lista RezervisaniBrojevi najverovatnije služi za rezervaciju delovodnih brojeva.
```

Status zaključka: **PRETPOSTAVKA**

Razlog:

- naziv liste ukazuje na rezervaciju brojeva
- polje `RezervisaniBroj` potvrđuje namenu
- ali nije dostavljena flow logika koja koristi ovu listu

---

## 3.2 Biblioteka `Shared Documents`

Scope:

```text
/sites/DocumentCentralv6.0/Shared Documents
```

Relevantno potvrđeno polje:

| Internal name | Display name | Tip | Status |
|---|---|---|---|
| `DelovodniBroj` | DelovodniBroj | Single line of text | Potvrđeno |

Zaključak:

```text
DelovodniBroj je finalna oznaka dokumenta ili jedan od ključnih metadata atributa registrovanog dokumenta.
```

Status zaključka: **DELIMIČNO POTVRĐENO**

---

# 4. Kritični poslovni zahtev

## 4.1 Pravilo

Sistem mora dozvoliti paralelni rad.

Primer:

```text
Korisnik A zavodi dokument u 10:00:01
Korisnik B zavodi dokument u 10:00:01
Korisnik C zavodi dokument u 10:00:01
```

Sva tri korisnika moraju moći da rade.

Ali rezultat mora biti:

```text
Korisnik A dobija broj 1001
Korisnik B dobija broj 1002
Korisnik C dobija broj 1003
```

Nikada ne sme biti:

```text
Korisnik A dobija broj 1001
Korisnik B dobija broj 1001
```

---

## 4.2 Zabranjeni model

Zabranjeno je da Canvas app sama radi sledeću logiku:

```text
1. Učitaj sve dokumente
2. Pronađi najveći DelovodniBroj
3. Dodaj 1
4. Patchuj dokument sa novim brojem
```

Razlog:

```text
Dva korisnika mogu u istom trenutku pročitati isti najveći broj i izračunati isti sledeći broj.
```

To je klasičan race condition.

---

## 4.3 Canvas app sme da radi

Canvas app sme da:

- prikaže formu
- validira obavezna polja
- pozove Power Automate flow
- prikaže korisniku rezultat
- prikaže grešku ako backend ne uspe
- osveži kolekcije nakon uspešnog zavođenja

Canvas app ne sme da:

- samostalno određuje finalni delovodni broj
- garantuje jedinstvenost
- piše direktno u kritična polja ako korisnici imaju Read Only prava
- radi lokalni `Max() + 1` kao izvor istine
- menja counter bez backend provere

---

# 5. Problem konkurentnosti

## 5.1 Race condition primer

Ako dva korisnika istovremeno pokrenu zavođenje:

```text
Trenutni poslednji broj = 1000
```

Bez locking-a:

```text
Korisnik A čita 1000
Korisnik B čita 1000

Korisnik A računa 1001
Korisnik B računa 1001

Korisnik A upisuje 1001
Korisnik B upisuje 1001
```

Rezultat:

```text
DUPLIKAT
```

---

## 5.2 Zašto samo provera pre upisa nije dovoljna

Čak i ako flow uradi proveru:

```text
Da li postoji DelovodniBroj = 1001?
```

To nije dovoljno ako posle toga nema atomic upisa.

Problem:

```text
Flow A proveri da 1001 ne postoji
Flow B proveri da 1001 ne postoji
Flow A upiše 1001
Flow B upiše 1001
```

Zaključak:

```text
Provera mora biti kombinovana sa unique constraint-om ili atomic/optimistic locking mehanizmom.
```

---

# 6. Preporučeni enterprise model

## 6.1 Osnovni princip

Jedini dozvoljeni izvor istine za broj treba da bude backend servis.

U Power Platform + SharePoint scenariju taj servis može biti:

```text
Power Automate flow + SharePoint counter/lock list + unique constraint
```

Ciljni princip:

```text
Canvas app šalje zahtev.
Power Automate generiše i rezerviše broj.
Power Automate upisuje dokument.
Power Automate vraća rezultat Canvas app-u.
```

---

## 6.2 Target sequence

```text
Canvas App
    |
    | 1. Submit registration request
    v
Power Automate: RegisterDocument
    |
    | 2. Validate input
    v
Power Automate: ReserveNumber
    |
    | 3. Acquire lock / optimistic update
    v
SharePoint: NumberCounter / RezervisaniBrojevi
    |
    | 4. Reserve unique number
    v
Power Automate: RegisterDocument
    |
    | 5. Create/update document metadata
    v
SharePoint: Shared Documents
    |
    | 6. Enforce unique DelovodniBroj
    v
Power Automate
    |
    | 7. Audit result
    v
Canvas App
```

---

# 7. Preporučena lista za counter

## 7.1 Nova lista `NumberCounters`

Za enterprise verziju preporučuje se posebna lista:

```text
NumberCounters
```

Svrha:

```text
Centralni brojač po delovodnoj knjizi, godini i eventualno tipu dokumenta.
```

Predložena polja:

| Polje | Tip | Opis |
|---|---|---|
| `Title` | Single line of text | Jedinstveni ključ counter-a |
| `CounterKey` | Single line of text | Npr. `OP-2026` |
| `DelovodnaKnjiga` | Single line of text / Lookup | Knjiga |
| `Godina` | Number | Godina |
| `Prefix` | Single line of text | Prefiks broja |
| `LastNumber` | Number | Poslednji dodeljeni broj |
| `Locked` | Yes/No | Da li je counter trenutno zaključan |
| `LockedBy` | Single line of text | Flow/user koji drži lock |
| `LockedAt` | Date and Time | Kada je lock postavljen |
| `LockExpiresAt` | Date and Time | Kada lock ističe |
| `ETagSnapshot` | Single line of text | Opcioni zapis ETag vrednosti |
| `Modified` | Date and Time | SharePoint sistemsko |
| `Editor` | Person or Group | SharePoint sistemsko |

Preporuka:

```text
CounterKey mora biti unique.
```

Primer:

```text
OP-2026
UG-2026
FAK-2026
```

---

## 7.2 Zašto ne koristiti samo `RezervisaniBrojevi` kao counter

`RezervisaniBrojevi` može biti korisna kao log rezervisanih brojeva, ali nije idealna kao jedini counter ako nema jasno:

- jedinstveni counter key
- ETag kontrolu
- status rezervacije
- expiration
- flow run id
- user id
- rollback/cleanup model

Zato se preporučuje:

```text
NumberCounters = trenutni brojač
RezervisaniBrojevi = log rezervacija
```

---

# 8. Preporučena proširena lista `RezervisaniBrojevi`

Trenutno potvrđena polja su minimalna.

Za enterprise verziju preporučuje se proširenje:

| Polje | Tip | Opis |
|---|---|---|
| `Title` | Single line of text | Human readable key |
| `RezervisaniBroj` | Number | Rezervisani broj |
| `FullDelovodniBroj` | Single line of text | Finalni format broja |
| `CounterKey` | Single line of text | Veza ka counter-u |
| `Godina` | Number | Godina |
| `DelovodnaKnjiga` | Single line of text | Knjiga |
| `DatumRezervacije` | Date and Time | Datum rezervacije |
| `ReservedByEmail` | Single line of text | Korisnik koji je pokrenuo |
| `ReservedByFlowRunId` | Single line of text | Flow run |
| `ReservationStatus` | Choice | Reserved/Committed/Expired/Failed/Cancelled |
| `CommittedDocumentId` | Number | ID dokumenta |
| `CommittedLibrary` | Single line of text | Biblioteka |
| `CommittedAt` | Date and Time | Kada je broj iskorišćen |
| `ErrorMessage` | Multiple lines of text | Greška |
| `RequestPayloadJson` | Multiple lines of text | Ulazni payload |
| `ResponsePayloadJson` | Multiple lines of text | Izlazni payload |

---

# 9. Preporučeni algoritam

## 9.1 Algoritam sa optimistic locking

Preporučeni model:

```text
1. Flow dobija zahtev iz Canvas app-a.
2. Flow validira input.
3. Flow učitava counter item za odgovarajući CounterKey.
4. Flow čita trenutni ETag counter item-a.
5. Flow izračuna NextNumber = LastNumber + 1.
6. Flow pokušava update counter item-a koristeći If-Match sa ETag vrednošću.
7. Ako update uspe:
   - broj je rezervisan
   - kreira se zapis u RezervisaniBrojevi
   - nastavlja se zavođenje dokumenta
8. Ako update ne uspe zbog ETag konflikta:
   - drugi flow je pre toga promenio counter
   - flow čeka kratko
   - ponovo čita counter
   - ponavlja pokušaj
9. Ako ni posle više pokušaja ne uspe:
   - vraća kontrolisanu grešku
   - auditira neuspeh
```

---

## 9.2 Pseudokod

```text
Input:
  CounterKey
  DocumentPayload
  RequestedByEmail

MaxRetries = 5
Attempt = 0

Do until Attempt >= MaxRetries:

  Attempt = Attempt + 1

  Get Counter item by CounterKey

  Read:
    LastNumber
    ETag

  NextNumber = LastNumber + 1
  FullDelovodniBroj = format(Prefix, Year, NextNumber)

  Try update Counter item:
    LastNumber = NextNumber
    If-Match = ETag

  If update success:
    Create reservation log
    Create/update document with FullDelovodniBroj
    Audit success
    Return success

  If update conflict:
    Delay random 200-1000ms
    Continue

Return failure:
  Audit failure
  Return controlled error
```

---

# 10. Power Automate implementacija

## 10.1 Glavni flow

Predloženi flow:

```text
DC_RegisterDocument
```

Trigger:

```text
Power Apps V2
```

Ulazni parametri:

| Parametar | Tip | Opis |
|---|---|---|
| `CounterKey` | Text | Ključ brojača |
| `DocumentJson` | Text | JSON dokumenta |
| `RequestedByEmail` | Text | Korisnik |
| `SourceLibrary` | Text | Izvor |
| `SourceItemId` | Text/Number | ID izvornog dokumenta |
| `OperationId` | Text | GUID iz Canvas app-a |

Izlaz:

| Polje | Tip | Opis |
|---|---|---|
| `success` | Boolean | Da li je uspelo |
| `delovodniBroj` | Text | Dodeljeni broj |
| `documentId` | Number | ID dokumenta |
| `reservationId` | Number | ID rezervacije |
| `errorCode` | Text | Kod greške |
| `errorMessage` | Text | Poruka greške |

---

## 10.2 Child flow za rezervaciju broja

Predloženi child flow:

```text
DC_ReserveNumber
```

Svrha:

- dobija `CounterKey`
- radi optimistic locking
- vraća jedinstven broj
- kreira reservation log

Ulaz:

```json
{
  "counterKey": "OP-2026",
  "requestedByEmail": "user@domain.com",
  "operationId": "guid"
}
```

Izlaz:

```json
{
  "success": true,
  "reservationId": 123,
  "reservedNumber": 1001,
  "fullDelovodniBroj": "OP-2026-001001"
}
```

---

# 11. SharePoint REST ETag update

## 11.1 Princip

Za optimistic locking koristi se SharePoint ETag.

Logika:

```text
Ako item nije promenjen od trenutka kada ga je flow pročitao,
update će uspeti.

Ako ga je drugi flow promenio,
update treba da padne sa conflict/precondition failed greškom.
```

---

## 11.2 REST koncept

Primer koncepta HTTP update-a:

```http
POST _api/web/lists/getbytitle('NumberCounters')/items(1)
IF-MATCH: "{etag vrednost}"
X-HTTP-Method: MERGE
Content-Type: application/json;odata=verbose
Accept: application/json;odata=verbose

{
  "__metadata": {
    "type": "SP.Data.NumberCountersListItem"
  },
  "LastNumber": 1001
}
```

Napomena:

Tačan `__metadata.type` mora se proveriti za konkretnu listu.

---

## 11.3 ETag conflict

Ako je counter item izmenjen između čitanja i update-a:

```text
Update ne sme biti tretiran kao fatalna greška odmah.
To je očekivani concurrency scenario.
```

Flow treba da:

1. uhvati grešku
2. poveća attempt counter
3. sačeka kratak random delay
4. ponovo pročita counter
5. pokuša ponovo

---

# 12. Unique constraint

## 12.1 Zašto je potreban

Optimistic locking smanjuje rizik, ali enterprise sistem treba dodatnu zaštitu.

Zato finalno polje:

```text
Shared Documents.DelovodniBroj
```

treba da ima unique constraint ako se koristi kao finalni jedinstveni broj.

To znači:

```text
Čak i ako backend logika pogreši, SharePoint treba da odbije duplikat.
```

---

## 12.2 Predlog

Za glavnu biblioteku:

```text
DelovodniBroj:
  EnforceUniqueValues = true
  Indexed = true
```

Napomena:

Pre uključivanja unique constraint-a treba proveriti:

- da li postoje postojeći duplikati
- da li polje sadrži prazne vrednosti
- da li je dozvoljeno više praznih vrednosti u konkretnom SharePoint ponašanju
- da li postoje dokumenti bez broja
- da li se isto polje koristi za draft dokumente

---

## 12.3 Ako postoje draft dokumenti bez broja

Ako `DelovodniBroj` nije poznat na početku procesa, moguća su dva modela.

### Model A — broj se dodeljuje tek pri finalnom zavođenju

Draft dokument nema `DelovodniBroj`.

Finalni dokument dobija `DelovodniBroj`.

Potrebno proveriti ponašanje unique constraint-a sa praznim vrednostima.

### Model B — koristiti posebno polje za finalni broj

Polja:

```text
DraftNumber
DelovodniBroj
```

`DelovodniBroj` se popunjava samo kada je dokument zaveden.

---

# 13. Retry logika

## 13.1 Retry mora biti kontrolisan

Retry ne sme biti beskonačan.

Preporuka:

```text
MaxRetries = 5
```

Delay:

```text
Random delay između 200ms i 1000ms
```

Kod većeg opterećenja:

```text
Exponential backoff
```

Primer:

| Attempt | Delay |
|---|---|
| 1 | 200–500 ms |
| 2 | 500–1000 ms |
| 3 | 1000–1500 ms |
| 4 | 1500–2500 ms |
| 5 | 2500–4000 ms |

---

## 13.2 Šta ako retry ne uspe

Flow mora da vrati kontrolisanu grešku:

```json
{
  "success": false,
  "errorCode": "NUMBER_RESERVATION_FAILED",
  "errorMessage": "Delovodni broj nije moguće rezervisati u ovom trenutku. Pokušajte ponovo."
}
```

I mora da upiše audit/error log.

---

# 14. Audit log

## 14.1 Obavezni audit događaji

Za numbering proces obavezno auditovati:

| Event | Opis |
|---|---|
| `NUMBER_RESERVATION_REQUESTED` | Korisnik je zatražio broj |
| `COUNTER_READ` | Counter je pročitan |
| `COUNTER_UPDATE_ATTEMPTED` | Pokušaj update-a |
| `COUNTER_UPDATE_CONFLICT` | ETag conflict |
| `NUMBER_RESERVED` | Broj rezervisan |
| `NUMBER_COMMITTED` | Broj upisan na dokument |
| `NUMBER_RESERVATION_FAILED` | Rezervacija nije uspela |
| `DUPLICATE_NUMBER_REJECTED` | Unique constraint odbio duplikat |
| `NUMBER_ROLLBACK_REQUIRED` | Potreban rollback/kompenzacija |

---

## 14.2 Predložena lista `AuditLog`

Polja:

| Polje | Tip | Opis |
|---|---|---|
| `Title` | Single line of text | Kratak opis |
| `EventType` | Choice | Tip događaja |
| `OperationId` | Single line of text | GUID operacije |
| `FlowRunId` | Single line of text | Flow run |
| `CounterKey` | Single line of text | Brojač |
| `ReservedNumber` | Number | Broj |
| `FullDelovodniBroj` | Single line of text | Finalni broj |
| `DocumentId` | Number | Dokument |
| `RequestedByEmail` | Single line of text | Korisnik |
| `Attempt` | Number | Pokušaj |
| `PayloadJson` | Multiple lines of text | Ulazni/izlazni payload |
| `ErrorCode` | Single line of text | Kod greške |
| `ErrorMessage` | Multiple lines of text | Poruka greške |
| `CreatedOn` | Date and Time | Datum |

---

# 15. Error handling

## 15.1 Standardni error response

Svi flow-ovi koji se pozivaju iz Canvas app-a treba da vraćaju standardizovan odgovor.

Primer uspeha:

```json
{
  "success": true,
  "operationId": "2a7b9b6d-7a2f-4aa7-b3df-0e613a948a12",
  "delovodniBroj": "OP-2026-001001",
  "documentId": 456,
  "reservationId": 123,
  "message": "Dokument je uspešno zaveden."
}
```

Primer greške:

```json
{
  "success": false,
  "operationId": "2a7b9b6d-7a2f-4aa7-b3df-0e613a948a12",
  "errorCode": "NUMBER_RESERVATION_FAILED",
  "errorMessage": "Delovodni broj nije moguće rezervisati u ovom trenutku. Pokušajte ponovo."
}
```

---

## 15.2 Error kodovi

Predloženi error kodovi:

| Error code | Opis |
|---|---|
| `INVALID_INPUT` | Nedostaju obavezni podaci |
| `COUNTER_NOT_FOUND` | Nije pronađen counter |
| `COUNTER_LOCK_TIMEOUT` | Lock nije oslobođen |
| `NUMBER_RESERVATION_FAILED` | Rezervacija broja nije uspela |
| `DUPLICATE_DELOVODNI_BROJ` | Detektovan duplikat |
| `DOCUMENT_UPDATE_FAILED` | Dokument nije ažuriran |
| `DOCUMENT_CREATE_FAILED` | Dokument nije kreiran |
| `AUDIT_WRITE_FAILED` | Audit nije upisan |
| `UNEXPECTED_ERROR` | Neočekivana greška |

---

# 16. Kompenzaciona logika

## 16.1 Problem

Može se desiti:

```text
Broj je rezervisan,
ali dokument nije uspešno kreiran/ažuriran.
```

Tada sistem ne sme tiho da ignoriše problem.

---

## 16.2 Preporučeno ponašanje

Ako je broj rezervisan, ali dokument nije zaveden:

1. Reservation status ostaje `Reserved` ili prelazi u `Failed`.
2. Audit log dobija grešku.
3. Broj se ne reciklira automatski bez poslovne odluke.
4. Admin može ručno da pregleda failed rezervaciju.
5. Korisnik dobija jasnu poruku.

Preporuka:

```text
Delovodni brojevi se ne recikliraju automatski.
```

Razlog:

- smanjuje se rizik od audit problema
- lakše se dokazuje šta se dogodilo
- rupe u numeraciji su prihvatljivije od duplikata u enterprise pisarnici

---

# 17. Lock model kao alternativa

## 17.1 Lock item model

Alternativno može postojati jedan lock item:

```text
Title = Lock
Value = true/false
```

Flow:

```text
1. Pokušaj da zauzme lock
2. Ako je lock slobodan, postavi lock
3. Generiši broj
4. Upisi dokument
5. Oslobodi lock
```

---

## 17.2 Rizici lock modela

Rizici:

- lock može ostati zauzet ako flow pukne
- potreban je lock timeout
- potreban je cleanup
- bottleneck kod velikog broja korisnika
- kompleksnije za više delovodnih knjiga

Ako se koristi lock model, obavezno:

- `LockExpiresAt`
- `LockedByFlowRunId`
- timeout provera
- retry
- audit
- admin unlock procedura

---

## 17.3 Preporuka

Za enterprise verziju preporuka je:

```text
Optimistic locking preko ETag-a + unique constraint na finalnom broju.
```

Lock item može biti dodatni mehanizam, ali ne treba da bude jedina zaštita.

---

# 18. Idempotency

## 18.1 Zašto je potrebno

Power Automate može biti pozvan više puta za isti korisnički zahtev zbog:

- double click
- retry iz Canvas app
- mrežni timeout
- korisnik ponovo klikne submit
- flow timeout posle uspešnog upisa

Bez idempotency modela, isti zahtev može dobiti dva broja.

---

## 18.2 OperationId

Canvas app treba da generiše:

```text
OperationId = GUID()
```

I da ga pošalje flow-u.

Flow mora da proveri:

```text
Da li već postoji uspešna rezervacija za isti OperationId?
```

Ako postoji:

- ne generiše novi broj
- vraća postojeći rezultat

---

## 18.3 Predlog

Dodati polje:

```text
OperationId
```

u:

- `RezervisaniBrojevi`
- `AuditLog`
- eventualno `Shared Documents`

I postaviti unique constraint ako je moguće.

---

# 19. Format delovodnog broja

## 19.1 Predloženi model

Delovodni broj treba formatirati centralno u backend-u.

Primer:

```text
{Prefix}-{Year}-{NumberPadded}
```

Primer:

```text
OP-2026-001001
```

---

## 19.2 Zašto format ne sme biti samo UI logika

Ako format radi samo Canvas app:

- flow može upisati drugačiji format
- integracije mogu upisati drugačije
- export može prikazati drugačije
- korisnici mogu videti nedoslednosti

Zato backend treba da vrati finalni format.

---

# 20. Preporučeni minimum za novu verziju

Minimalni enterprise paket za numbering:

1. `NumberCounters` lista.
2. `RezervisaniBrojevi` kao reservation log.
3. `AuditLog` lista.
4. `OperationId` za idempotency.
5. `CounterKey` za delovodnu knjigu/godinu.
6. ETag optimistic locking.
7. Retry sa random/exponential delay.
8. Unique constraint na finalni `DelovodniBroj`.
9. Standardizovan JSON response.
10. Kontrolisane greške.
11. Admin review za failed rezervacije.
12. Pravilo da se brojevi ne recikliraju automatski.

---

# 21. Predloženi flow dizajn

## 21.1 Flow: `DC_RegisterDocument`

Odgovornost:

- glavni flow koji poziva Canvas app
- validacija ulaza
- poziv child flow-a za broj
- kreiranje/ažuriranje dokumenta
- audit
- response Canvas app-u

---

## 21.2 Flow: `DC_ReserveNumber`

Odgovornost:

- generisanje broja
- optimistic locking
- reservation log
- retry
- response glavnom flow-u

---

## 21.3 Flow: `DC_CommitReservation`

Odgovornost:

- označavanje rezervacije kao iskorišćene
- povezivanje sa dokumentom
- upis `CommittedDocumentId`
- audit

---

## 21.4 Flow: `DC_CleanupExpiredReservations`

Odgovornost:

- pronalazi stare rezervacije koje nisu commitovane
- označava ih kao `Expired` ili `Failed`
- ne reciklira automatski broj
- šalje admin izveštaj

---

# 22. Test scenariji

## 22.1 Paralelno zavođenje

Test:

```text
10 korisnika istovremeno klikne "Zavedite dokument".
```

Očekivanje:

```text
Svaki korisnik dobija različit delovodni broj.
Nema duplikata.
Svi pokušaji su auditovani.
```

---

## 22.2 Double click

Test:

```text
Jedan korisnik dva puta brzo klikne submit.
```

Očekivanje:

```text
Ako je isti OperationId, sistem vraća isti rezultat.
Ne generiše se drugi broj.
```

---

## 22.3 ETag conflict

Test:

```text
Dva flow-a čitaju isti counter.
Jedan uspeva, drugi dobija conflict.
```

Očekivanje:

```text
Drugi flow radi retry i dobija sledeći broj.
```

---

## 22.4 Unique constraint collision

Test:

```text
Namerno pokušati upis istog DelovodniBroj.
```

Očekivanje:

```text
SharePoint odbija duplikat.
Flow vraća kontrolisanu grešku.
Audit log beleži incident.
```

---

## 22.5 Flow failure posle rezervacije

Test:

```text
Broj je rezervisan, ali update dokumenta ne uspe.
```

Očekivanje:

```text
Rezervacija ostaje auditovana.
Status prelazi u Failed ili Reserved.
Broj se ne reciklira automatski.
Admin može da vidi problem.
```

---

# 23. Open questions

1. Da li trenutno postoji flow koji koristi `RezervisaniBrojevi`?
2. Da li `RezervisaniBrojevi` već ima iteme?
3. Da li `RezervisaniBroj` mora biti jedinstven po godini?
4. Da li postoje različite delovodne knjige?
5. Da li format broja zavisi od tipa dokumenta?
6. Da li format broja zavisi od organizacione jedinice?
7. Da li `DelovodniBroj` trenutno ima unique constraint?
8. Da li korisnici direktno pišu u SharePoint ili sve ide kroz flow?
9. Da li se dokument prvo uploaduje, pa se zatim zavodi?
10. Da li se broj rezerviše pre ili posle upload-a dokumenta?
11. Da li neuspešno rezervisani brojevi smeju da se ponovo koriste?
12. Da li je prihvatljivo da postoje rupe u numeraciji?
13. Da li postoji zakonski ili interni zahtev za redosled bez rupa?
14. Da li postoji audit log danas?
15. Da li postoji admin ekran za greške?

---

# 24. Zaključak

Generisanje delovodnog broja je centralni enterprise rizik DocCentral rešenja.

Potvrđeno je da postoje:

- `RezervisaniBrojevi`
- `RezervisaniBroj`
- `DatumRezervacije`
- `Shared Documents.DelovodniBroj`

Nije potvrđeno da trenutno postoji dovoljno jak concurrency model.

Za novu verziju je obavezno projektovati numbering kao backend servis, ne kao Canvas app logiku.

Najbezbedniji preporučeni model:

```text
Power Automate backend
+ NumberCounters lista
+ ETag optimistic locking
+ RezervisaniBrojevi reservation log
+ unique constraint na DelovodniBroj
+ retry
+ OperationId idempotency
+ AuditLog
+ kontrolisane greške
```

Prioritet:

```text
P0 — bez ovoga sistem nije enterprise-ready.
```

---

## 25. Sledeći dokument

Sledeći dokument:

```text
07-power-apps-analysis.md
```

Taj dokument treba da obradi:

- pretpostavljenu Canvas app strukturu
- odgovornosti Canvas aplikacije
- šta aplikacija sme i ne sme da radi
- učitavanje konfiguracije iz `AppConfig`
- interakciju sa Power Automate flow-ovima
- preporuke za novu responzivnu, višejezičnu i brzu Canvas app verziju
