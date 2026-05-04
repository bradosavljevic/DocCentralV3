# Skill: Power Apps Responsive UI

## Cilj

Nova Canvas aplikacija mora biti potpuno responzivna, pogodna za desktop, laptop i veće tablete.

## Pravila

- Koristiti auto-layout kontejnere gde god je moguće.
- Izbegavati fiksne X/Y pozicije osim kada je opravdano.
- Koristiti skalabilne širine i visine: `Parent.Width`, `Parent.Height`, relativne formule i layout kontejnere.
- Ekrani moraju raditi u različitim rezolucijama.
- Komponente treba praviti modularno: header, sidebar, command bar, filter panel, status badge, form section, document card.

## Preporučeni layout

- Desktop: sidebar + content area.
- Tablet: collapsible sidebar + content.
- Mobilni prikaz nije primarni zahtev, ali layout ne sme potpuno pucati.

## UX principi

- Najvažnije akcije moraju biti vidljive: Novi predmet, Zavedi, Sačuvaj, Arhiviraj, Otkaži.
- Destruktivne akcije moraju imati potvrdu.
- Status flow operacija mora biti jasan korisniku.
- Spinner/loading state mora postojati kod svih flow poziva.
- Greške iz Power Automate-a moraju biti prikazane korisniku razumljivo.

## Stil

- Koristiti Fluent-like stil.
- Kontrast mora biti dovoljan.
- Statusi treba da imaju konzistentne boje.
- Dugi tekstovi i nazivi dokumenata moraju imati kontrolisano prelamanje ili tooltip.
