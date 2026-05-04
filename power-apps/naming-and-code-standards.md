# Power Apps naming i code standards

## Naming konvencija

Global variables:

- `gblVariableName`

Local/context variables:

- `locVariableName`

Collections:

- `colCollectionName`

Named formulas:

- `nfFormulaName`

Controls:

- screen: `scrName`
- container: `cntName`
- gallery: `galName`
- button: `btnName`
- label: `lblName`
- text input: `txtName`
- combo box: `cmbName`
- date picker: `dpName`
- form: `frmName`
- icon: `icoName`

## Error handling

Koristiti:

- `IfError`
- `Notify`
- kontrolisane error poruke
- logovanje greške kroz flow gde je potrebno

## Performance

Obavezno:

- delegabilne formule
- izbegavati live search na svaki karakter
- koristiti search button ili debounce pattern
- izbegavati cross-screen control references
- izbegavati nested galleries osim ako je opravdano
- minimalizovati broj poziva u OnVisible
- koristiti keširane konfiguracije gde je bezbedno

## App Checker

Aplikacija mora biti bez kritičnih App Checker grešaka.

## Multi-language

Aplikacija treba da podrži srpski i engleski.

Tekstove ne hardkodovati ako postoji konfiguracioni/translation layer.

## Locale

Podržati srpski format datuma i brojeva.
