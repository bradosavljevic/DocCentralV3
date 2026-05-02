# AppConfig — AppLock

## Svrha

Konfiguracija zaključavanja aplikacije za sprečavanje paralelnih kritičnih operacija.

## Izvor

- SharePoint lista: `AppConfig`
- Stavka: `AppLock`
- SharePoint item ID: `23`
- Poslednja izmena: `4.2.2026. 21:59`
- Deklarisani `ColumnHeader`: ``
- Broj zapisa u JSON konfiguraciji: `1`

## Struktura JSON zapisa

| Polje | Napomena |
| --- | --- |
| `AppLock` | Polje je pronađeno u JSON konfiguraciji. |
| `Title` | Polje je pronađeno u JSON konfiguraciji. |
| `LockedBy` | Polje je pronađeno u JSON konfiguraciji. |
| `LockedAt` | Polje je pronađeno u JSON konfiguraciji. |
| `LockToken` | Polje je pronađeno u JSON konfiguraciji. |

## Podaci

| AppLock | Title | LockedBy | LockedAt | LockToken |
| --- | --- | --- | --- | --- |
| False | AppLock |  |  |  |

## Činjenice

- Konfiguracija je čuvana kao JSON tekst u koloni `Config`.
- Stavka sadrži `1` JSON zapisa.
- Pronađena polja: `AppLock`, `Title`, `LockedBy`, `LockedAt`, `LockToken`.

## Nepoznato / potrebno potvrditi

- Da li Power Automate koristi ovu konfiguraciju kao jedini izvor istine u trenutku upisa.
- Da li postoji atomic/optimistic locking i retry logika.

## Preporuke za novu verziju

- Validirati JSON šemu pre učitavanja konfiguracije u aplikaciju ili flow.
- Dodati verzionisanje konfiguracije ako se ista menja kroz administrativni ekran.
- Za kritične konfiguracije dodati audit trag: ko je promenio, kada, staru vrednost i novu vrednost.
- Ako konfiguracija utiče na prava pristupa, ne oslanjati se samo na UI filtriranje; obavezno uskladiti sa SharePoint/Entra ID permission modelom.
