# 01 - Business Overview

DocCentral / e-pisarnica je Power Platform rešenje za kontrolisano zavođenje, obradu, odobravanje, arhiviranje i zaključavanje poslovnih dokumenata.

## Poslovna svrha

Aplikacija omogućava:
- unos novog dokumenta
- dodelu delovodnog broja
- rad sa rezervisanim delovodnim brojevima
- upload dokumenta i priloga
- obradu dokumenata pristiglih emailom
- proces odobravanja
- arhiviranje dokumenata
- generisanje arhivske knjige
- zaključavanje godine
- administraciju šifarnika i konfiguracija
- rad sa partnerima

## Poslovni princip

Aplikacija nije primarni viewer dokumenata. Aplikacija služi za kontrolisanu manipulaciju dokumentima, metapodacima i procesima. Pregled dokumenata i informacija se u osnovi radi direktno kroz SharePoint liste i biblioteke.

## Ključni entiteti

| Entitet | Opis |
|---|---|
| Svi predmeti | Glavna lista za metadata dokumenata |
| Dokumenta | Glavna biblioteka za fizičke dokumente i priloge |
| EmailDocuments | Staging biblioteka za dokumente pristigle emailom |
| AppConfig | Centralni JSON configuration store |
| Rezervisani brojevi | Lista rezervisanih delovodnih brojeva |
| Partneri | Lista poslovnih partnera |
| AuditLog | Log/history lista |
| Podsetnici | Lista podsetnika |

## Najvažnije poslovno pravilo

Delovodni broj je kritičan. Brojevi se ne smeju duplirati, preskakati ili ručno zaobilaziti u slučaju greške. Izuzetak su eksplicitno rezervisani brojevi.
