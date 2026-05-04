# Power Automate flow inventory - plan

## Cilj

Definisati flow-ove potrebne za novu verziju DocCentral V3.

## Predloženi flow-ovi

### 1. RegisterDocument

Svrha:

- validacija zahteva
- generisanje ili korišćenje rezervisanog delovodnog broja
- kreiranje dokument item-a
- kreiranje foldera/biblioteke
- dodela prava
- audit log
- odgovor Power Apps aplikaciji

### 2. GenerateOrReserveRegistryNumber

Svrha:

- centralizovano concurrency-safe generisanje broja
- ETag / optimistic locking
- retry logika
- logovanje pokušaja

### 3. UseReservedNumber

Svrha:

- validacija rezervisanog broja
- korišćenje rezervisanog broja
- brisanje nakon uspešnog zavođenja

### 4. StartApprovalProcess

Svrha:

- pokretanje approval toka
- slanje korisniku ili grupi
- status `U odobravanju`

### 5. ProcessApprovalResponse

Svrha:

- upis rezultata odobravanja
- promena polja `Stanje`
- obrada odbijanja
- vraćanje inicijatoru

### 6. SendDailyReminders

Svrha:

- scheduled flow u 08:00
- pronalazak današnjih podsetnika
- slanje email-a
- sprečavanje duplog slanja
- logovanje grešaka

### 7. ArchiveDocuments

Svrha:

- arhiviranje dokumenata iz statusa `Zavedeno`
- dodela arhivskih znakova
- promena statusa u `Arhivirano`
- logovanje

### 8. CloseRegistryYear

Svrha:

- proverava da su svi dokumenti arhivirani
- proverava da nema rezervisanih brojeva
- zaključava godinu
- kreira novu delovodnu knjigu
- menja aktivnu godinu u App Config
- loguje rezultat

### 9. ExportConfigurationToExcel

Svrha:

- export jednog šifarnika
- export svih šifarnika

### 10. GenerateArchiveBookPdf

Svrha:

- generisanje PDF Arhivske knjige
- čuvanje PDF-a u odgovarajuću biblioteku/folder
- logovanje

## Obavezni standardi

- svaki kritičan flow mora imati error handling
- svaki kritičan flow mora pisati log
- koristiti correlation id
- vraćati kontrolisan response u Power Apps
- Create/Edit/Delete u SharePoint raditi pod servisnim nalogom
