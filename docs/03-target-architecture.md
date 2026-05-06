# 03 - Target Architecture

## Target model

```text
Canvas App
    |
    | read: SharePoint or Power Automate read flows
    | write: Power Automate only
    |
Power Automate
    |
    | service account
    |
SharePoint Online
```

## Canvas app responsibilities

Canvas app radi UI, responzivni layout, lokalnu validaciju, učitavanje konfiguracija, poziv Power Automate flow-ova, prikaz uspeha/greške i kreiranje kolekcija od JSON response-a.

Canvas app ne radi direktan Patch u SharePoint, direktan Remove iz SharePoint-a, direktno povećanje delovodnog broja, direktno ažuriranje AppConfig-a ili dodelu finalnog delovodnog broja.

## Power Automate responsibilities

Power Automate radi create/update/delete operacije, dodelu delovodnog broja, upload/copy fajlova, update metapodataka, AppConfig update, AuditLog, recovery operacije, large-list read operacije i cleanup operacije.

## Human-in-the-loop

Claude Code/MCP Canvas ne kreira backend komponente. Za svaku novu backend komponentu Claude mora prvo napisati implementacioni dokument. Developer zatim ručno kreira komponentu i vraća tehničke detalje.
