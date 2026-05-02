# SharePoint exports i konfiguracioni podaci

## 1. Svrha dokumenta

Ovaj dokument opisuje kako se u projektu DocCentral koriste SharePoint REST exporti za dokumentovanje data modela.

Exporti su potrebni da bi se tačno dokumentovali:

- liste;
- biblioteke;
- kolone;
- interni nazivi;
- tipovi podataka;
- lookup veze;
- choice vrednosti;
- indexed / required / unique podešavanja;
- hidden i read-only polja.

## 2. Zašto su exporti važni

Bez REST exporta nije moguće 100% pouzdano znati:

- interne nazive kolona;
- da li je polje Choice, Text, Lookup ili Person;
- da li je kolona required;
- da li je kolona indeksirana;
- da li je uključena unikatnost;
- koje su stvarne Choice vrednosti;
- koje lookup liste i lookup fieldovi se koriste.

Zbog toga dokumentacija mora jasno razdvajati:

- poznate činjenice;
- pretpostavke;
- nepoznato.

## 3. Preporučeni REST endpointi

### 3.1 Pregled svih lista na site-u

```text
https://<tenant>.sharepoint.com/sites/<site>/_api/web/lists?$select=Title,Id,RootFolder/Name,RootFolder/ServerRelativeUrl,Hidden,BaseTemplate&$expand=RootFolder
```

### 3.2 Pregled svih kolona određene liste

```text
https://<tenant>.sharepoint.com/sites/<site>/_api/web/lists/getbytitle('<LIST_NAME>')/fields?$select=Title,InternalName,TypeAsString,Required,ReadOnlyField,Hidden,Indexed,EnforceUniqueValues,DefaultValue
```

### 3.3 Pregled kolona sa Choice vrednostima

```text
https://<tenant>.sharepoint.com/sites/<site>/_api/web/lists/getbytitle('<LIST_NAME>')/fields?$select=Title,InternalName,TypeAsString,Choices
```

### 3.4 Pregled lookup konfiguracije

```text
https://<tenant>.sharepoint.com/sites/<site>/_api/web/lists/getbytitle('<LIST_NAME>')/fields?$select=Title,InternalName,TypeAsString,LookupList,LookupField
```

### 3.5 Pregled konkretne liste po GUID-u

```text
https://<tenant>.sharepoint.com/sites/<site>/_api/Web/Lists(guid'<LIST_GUID>')/Fields?$select=Title,InternalName,TypeAsString,Required,ReadOnlyField,Hidden,Indexed,EnforceUniqueValues,DefaultValue
```

## 4. Liste / biblioteke koje treba dokumentovati

| Objekat | Tip | Status dokumentacije |
|---|---|---|
| Svi predmeti | SharePoint list | Delimično dokumentovano. Potreban pun export. |
| Partneri | SharePoint list | Delimično dokumentovano. Potreban pun export. |
| App Config | SharePoint list | Delimično dokumentovano. Potreban pun export. |
| Dokumenta / Shared Documents | Document library | Delimično dokumentovano. Potreban pun export. |
| Email Documents | List / Library | Nepoznat tip. Potreban export. |
| Rezervisani brojevi | SharePoint list | Potencijalna / preporučena lista. Potvrditi postojanje. |

## 5. Preporučeni format čuvanja exporta u GitHub-u

Predložena struktura:

```text
exports/
  sharepoint/
    lists-index.xml
    svi-predmeti-fields.xml
    partneri-fields.xml
    app-config-fields.xml
    dokumenta-fields.xml
    email-documents-fields.xml
```

Ako export dolazi kao JSON:

```text
exports/
  sharepoint/
    lists-index.json
    svi-predmeti-fields.json
    partneri-fields.json
    app-config-fields.json
    dokumenta-fields.json
```

## 6. Pravilo dokumentovanja

Svaki `.md` fajl u `data-model` folderu treba ažurirati na osnovu exporta.

Za svaku kolonu dokumentovati:

- display name;
- internal name;
- type;
- required;
- hidden;
- read only;
- indexed;
- unique;
- lookup target;
- choice vrednosti;
- poslovnu ulogu;
- gde se koristi u Power Apps;
- gde se koristi u Power Automate.

## 7. Minimalna komanda za proveru statusa export fajlova

```bash
find exports -type f | sort
```

## 8. Poznate nepoznanice

- Da li svi exporti postoje lokalno.
- Da li su exporti XML ili JSON.
- Da li su exporti iz produkcije, testa ili development okruženja.
- Da li postoje razlike između okruženja.
- Da li su connection reference i environment variable vrednosti posebno eksportovane.
