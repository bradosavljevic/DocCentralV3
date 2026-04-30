# DocCentral v6.0 — Tehnička dokumentacija i razvojna osnova

## 1. Svrha dokumentacije

Ovaj repozitorijum sadrži tehničku dokumentaciju za Power Platform rešenje **DocCentral v6.0**.

Cilj dokumentacije je:

1. Razumevanje postojeće aplikacije.

2. Dokumentovanje Power Apps, Power Automate i SharePoint data modela.

3. Identifikovanje tehničkog duga i rizika.

4. Priprema osnove za razvoj nove enterprise verzije aplikacije.

5. Priprema razvojnih instrukcija za Cloud Code / Claude Code / AI-assisted development.

---

## 2. Opis rešenja

DocCentral v6.0 je Power Platform rešenje za elektronsku pisarnicu / centralnu evidenciju dokumenata.

Na osnovu dostavljenih SharePoint REST XML metadata odgovora, rešenje koristi SharePoint Online kao primarni data layer i storage layer.

Identifikovane komponente:

- SharePoint liste

- SharePoint document libraries

- Power Apps Canvas aplikacija

- Power Automate flow-ovi

- konfiguraciona lista `AppConfig`

- lista za rezervaciju brojeva `RezervisaniBrojevi`

- biblioteke dokumenata

- email dokument biblioteka

- export biblioteka

---

## 3. Identifikovane SharePoint lokacije

Site:

```text

https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0

## REST API base:
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/


