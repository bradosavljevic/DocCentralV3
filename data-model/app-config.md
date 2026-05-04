# SharePoint lista: App Config

## 1. Svrha liste

Lista **App Config** služi kao centralna konfiguraciona lista / šifarnik za rešenje **DocCentral**.

U njoj se nalaze vrednosti koje Canvas Power Apps aplikacija i Power Automate flow-ovi koriste za prikaz, filtriranje, ponašanje aplikacije i poslovnu logiku.

## 2. Uloga u rešenju

**Poznato:**

- Lista sadrži šifarnike i konfiguracije.
- Koristi se kao izvor vrednosti za aplikaciju.
- Može sadržati vrednosti za dropdown / combobox kontrole.
- Može sadržati poslovne parametre koji ne treba da budu hardkodovani u aplikaciji.
- Deo je SharePoint data modela za DocCentral.

**Nepoznato:**

- Kompletan spisak kolona nije potvrđen bez punog exporta.
- Tačna struktura zapisa nije potvrđena.
- Nisu potvrđene sve kategorije konfiguracija.

## 3. Preporučena struktura konfiguracione liste

Ako lista već ne sadrži sve ove kolone, preporuka je da se App Config standardizuje približno ovako:

| Kolona | Tip | Svrha |
|---|---|---|
| Title | Single line of text | Naziv konfiguracione vrednosti. |
| ConfigKey | Single line of text | Tehnički ključ konfiguracije. |
| ConfigValue | Single line of text / Multiple lines | Vrednost konfiguracije. |
| ConfigGroup | Choice / Text | Grupa konfiguracije, npr. DocumentTypes, InvoiceTypes, UI, Security. |
| SortOrder | Number | Redosled prikaza. |
| IsActive | Yes/No | Da li je vrednost aktivna. |
| Language | Choice / Text | Jezik vrednosti, npr. sr, en. |
| Description | Multiple lines of text | Opis konfiguracije. |

## 4. Moguće kategorije konfiguracija

Na osnovu poznate logike DocCentral rešenja, App Config može sadržati sledeće grupe:

| Kategorija | Opis |
|---|---|
| DocumentTypes | Tipovi dokumenata. |
| InvoiceTypes | Vrste faktura. |
| TaxStatus | Poreski statusi. |
| OriginalCopy | Vrednosti za original/kopija/mail. |
| ApprovalRoles | Role i odobravaoci. |
| UI | UI podešavanja, tekstovi, default vrednosti. |
| Localization | Prevodi za srpski i engleski. |
| FlowConfig | Parametri koje koriste flow-ovi. |
| SecurityGroups | Grupe koje aplikacija koristi za prava i filtriranje. |
| Numbering | Pravila za delovodne brojeve. |

## 5. Poznate vrednosti koje mogu pripadati App Config listi

### VrstaFakture

- Izlazne fakture
- Izlazni avansni računi
- Ulazne fakture
- Ulazni avansni računi
- Ulazno knjižno odobrenje
- Ulazno knjižno zaduženje
- Izlazno knjižno odobrenje
- Izlazno knjižno zaduženje

### Vrsta

- SEF
- Fiskal
- Običan

### OrginalKopija

- Mail
- Orginal
- Kopija

### Statusi / Stanja

Poznato iz procesa:

- Zavedeno
- Arhivirano

Ostale vrednosti su `NEPOZNATO` dok se ne potvrde iz exporta.

## 6. Korišćenje u Power Apps

Preporučeni obrazac:

1. Na startu aplikacije ili pri ulasku u ekran pozvati flow koji čita App Config.
2. Flow vraća JSON sa aktivnim konfiguracijama.
3. Power Apps kreira kolekciju, npr. `colAppConfig`.
4. Kontrole koriste Filter nad kolekcijom po grupi i aktivnom statusu.

Primer konceptualne logike:

```powerfx
ClearCollect(
    colAppConfig,
    Table(ParseJSON(flowGetAppConfig.Run().result))
)
```

Napomena: konkretan Power Fx kod zavisi od stvarnog JSON odgovora flow-a.

## 7. Korišćenje u Power Automate

Power Automate flow-ovi mogu koristiti App Config za:

- mapiranje tipova dokumenata;
- definisanje pravila numeracije;
- definisanje grupa za prava pristupa;
- definisanje email primaoca;
- podešavanje ponašanja bez izmene flow-a;
- uključivanje / isključivanje delova procesa.

## 8. Enterprise preporuke

- Konfiguracione ključeve ne hardkodovati u više flow-ova bez centralne definicije.
- Dodati `IsActive` polje ako ne postoji.
- Dodati `SortOrder` za UI prikaz.
- Dodati `Environment` ako se ista lista koristi za Dev/Test/Prod logiku.
- Dodati audit izmene konfiguracija.
- Ograničiti write prava nad App Config listom.
- Razmotriti posebnu listu za prevode ako konfiguracija postane prevelika.

## 9. Poznate nepoznanice

- Tačne kolone App Config liste.
- Da li postoji grupisanje konfiguracija.
- Da li postoji `IsActive`.
- Da li postoji `SortOrder`.
- Da li postoje višejezične vrednosti.
- Da li flow-ovi direktno čitaju App Config ili samo Power Apps.
- Da li postoji environment-specific konfiguracija.
