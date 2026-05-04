# Arhitektura rešenja

## Platforma

- Microsoft Power Apps Canvas
- Microsoft Power Automate
- SharePoint Online
- Microsoft 365 / Office 365 Users
- Microsoft 365 Groups / Entra ID
- App Config SharePoint lista
- SharePoint liste za audit/logging
- SharePoint biblioteke/folderi za dokumente

## Osnovni obrazac

Canvas aplikacija je frontend.

Power Automate je business/API layer.

SharePoint Online je data/document layer.

## Pristup podacima

Read:

- Canvas app može čitati SharePoint liste gde korisnik ima pravo čitanja.
- Canvas app može čitati Office 365 Users i Microsoft 365 Groups podatke.
- Canvas app može čitati konfiguracije iz App Config.

Write:

- Create/Edit/Delete operacije moraju ići kroz Power Automate.
- Flow radi pod servisnim nalogom.
- Korisnici nemaju direktna Write prava nad SharePoint listama.

## Permission model

- Service account: Read/Write
- Owners: Read/Write
- Members: Read
- Viewers: Read
- End users: Read Only nad relevantnim SharePoint lokacijama i item-ima
- Item/folder/file permissions se dodeljuju kroz break inheritance

## White-label model

Rešenje se prilagođava po klijentu kroz:

- App Config
- ProcesConfig
- šifarnike
- Entra group mapping
- organizacione jedinice
- tekstove i prevode
- tipove dokumenata
- statusne tokove

## Enterprise zahtevi

- concurrency-safe generisanje delovodnog broja
- audit/logging kroz SharePoint liste
- retry logic za kritične flow-ove
- error handling u Power Apps i Power Automate
- responzivan Canvas app
- delegabilne formule
- test matrica po scenarijima
