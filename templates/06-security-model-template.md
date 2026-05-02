# Security and Permission Model

## Princip

Opisati kako korisnici pristupaju aplikaciji, SharePoint listama i dokumentima.

## Korisničke role

| Rola | Opis | SharePoint prava | Power Apps prava | Power Automate uticaj |
|---|---|---|---|---|
| NEPOZNATO | NEPOZNATO | NEPOZNATO | NEPOZNATO | NEPOZNATO |

## Service account

| Polje | Vrednost |
|---|---|
| Account | NEPOZNATO |
| Svrha | NEPOZNATO |
| Licence | NEPOZNATO |
| Vlasnik flow-ova | NEPOZNATO |
| SharePoint prava | NEPOZNATO |

## Grupe

| Grupa | Tip | Svrha | Članstvo | Napomena |
|---|---|---|---|---|
| NEPOZNATO | NEPOZNATO | NEPOZNATO | NEPOZNATO | NEPOZNATO |

## Rizici

- Direktan write pristup korisnika nad SharePoint listama ako postoji.
- Nejasno vlasništvo nad flow-ovima.
- Nejasan lifecycle service account-a.
- Nedostatak audit loga za kritične operacije.

## Preporuke

- Korisnicima dati Read Only gde je moguće.
- Upise izvršavati preko Power Automate service account-a.
- Dokumentovati sve grupe i role.
- Uvesti audit i error log.
