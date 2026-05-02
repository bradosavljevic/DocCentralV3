# AppConfig dokumentacija

Ovaj folder opisuje SharePoint listu `AppConfig`, u kojoj se nalaze šifarnici, konfiguracije procesa, prevodi, podešavanja, parametri e-faktura i tehnički statusi aplikacije.

## Pregled konfiguracionih stavki

| Konfiguracija | MD fajl | Broj zapisa | Poslednja izmena | Glavna polja |
| --- | --- | ---: | --- | --- |
| `ArhivskaKnjigaGrupeKategorija` | `data-model/app-config/arhivskaknjigagrupekategorija.md` | 11 | 5.8.2025. 10:30 | Title, RedniBroj |
| `ArhivskaListaKategorije` | `data-model/app-config/arhivskalistakategorije.md` | 247 | 29.8.2025. 10:53 | RedniBroj, Title, GrupaKategorija, KlasifikacionaOznaka, PoslednjiBrojRegistratruneJedinice, RokCuvanja, BrojMeseci, Opis |
| `KategorijeDokumentarnogMaterijala` | `data-model/app-config/kategorijedokumentarnogmaterijala.md` | 248 | 29.5.2025. 10:20 | Title, AKKategorija, AKGrupaKategorije, PoslednjiBrojRegistraturneJedinice, PoslednjaKoriscenaLokacija, KlasifikacionaOznaka |
| `DelovodneKnjige` | `data-model/app-config/delovodneknjige.md` | 2 | 29.4.2026. 09:01 | PoslednjiDelovodniBroj, PrefiksZaDelovodniBroj, Title, Godina |
| `OrganizacioneJedinice` | `data-model/app-config/organizacionejedinice.md` | 15 | 23.4.2026. 14:02 | MenadzerSektora, MenadzerSektoraEmail, Title, ZamenikMenadzerSektora, ZamenikMenadzerSektoraEmail |
| `LokacijeRegistracionihJedinica` | `data-model/app-config/lokacijeregistracionihjedinica.md` | 3 | 22.10.2025. 17:20 | Title |
| `Permisije` | `data-model/app-config/permisije.md` | 2 | 24.6.2025. 10:44 | DocumentCategory, DocumentType, Group, GroupEmail, InputOutput, OrganizationalUnit, Title |
| `RokoviCuvanja` | `data-model/app-config/rokovicuvanja.md` | 12 | 29.5.2025. 10:20 | Title, BrojMeseci, TrajnoCuvanje |
| `Stanja` | `data-model/app-config/stanja.md` | 11 | 24.6.2025. 15:31 | Title, TipDokumenta, DozvoliArhiviranje, OmoguciPromenuStanja, PocetnoStanje, KrajnjeStanje, IsProces, ApprovalType, Person, PearsonEmail, Treshold, RedolsedStanjaTreshold, RedosledStanja |
| `TipoviDokumenta` | `data-model/app-config/tipovidokumenta.md` | 45 | 27.9.2025. 11:53 | Title, VrstaDokumenta |
| `TipRegistracioneJedinice` | `data-model/app-config/tipregistracionejedinice.md` | 3 | 5.8.2025. 10:32 | Title |
| `Translations` | `data-model/app-config/translations.md` | 193 | 25.2.2026. 12:25 | Title, Lang, Value |
| `Users` | `data-model/app-config/users.md` | 3 | 21.4.2026. 11:47 | Lang, Rola, Title, User |
| `Valute` | `data-model/app-config/valute.md` | 10 | 22.7.2025. 07:35 | Title, Treshold |
| `VrsteDokumenta` | `data-model/app-config/vrstedokumenta.md` | 6 | 27.9.2025. 11:50 | Title |
| `RegistracioneJedinice` | `data-model/app-config/registracionejedinice.md` | 6 | 27.2.2026. 10:05 | Title, Lokacija, TipRegistracioneJedinice, BrojRegistracioneJedinice, GodinaNastanka, KlasifikacionaOznaka, AKKategorija, AKKategorijaGrupaKategorija |
| `EFaktureParametri` | `data-model/app-config/efaktureparametri.md` | 4 | 23.6.2025. 17:25 | Title, SifraDokumenta, VrstaDokumenta, TipDokumenta, DelovodnaKnjiga, AKGrupaKategorije, AKKategorija, AKKategorijaKlasifikacionaOznaka |
| `Settings` | `data-model/app-config/settings.md` | 1 | 10.6.2025. 13:21 | SPLink, Proces, AdminNalog |
| `ProcesConfig` | `data-model/app-config/procesconfig.md` | 6 | 12.8.2025. 12:26 | TipDokumenta, Iznos, RedosledKoraka, RedosledKorakaTreshold, NacinOdobravanja, OdobravalacGrupa, NazivKoraka, Aktivan, KomentarObavezan, OrganizacionaJedinica, Napomena |
| `EfaktureParametri` | `data-model/app-config/efaktureparametri.md` | 4 | 4.8.2025. 14:04 | Naziv dokumenta, Šifra dokumenta, Vrsta dokumenta, Tip dokumenta, Delovodnik, AK grupa kategorije, AK Kategorija, AK Klasifikaciona oznaka |
| `AppLock` | `data-model/app-config/applock.md` | 1 | 4.2.2026. 21:59 | AppLock, Title, LockedBy, LockedAt, LockToken |
| `SEFDCDatumPovlacenja` | `data-model/app-config/sefdcdatumpovlacenja.md` | 1 | 21.4.2026. 16:02 | LastDate |

## Ključni zaključci

- `AppConfig` nije običan šifarnik već centralna konfiguraciona lista aplikacije.
- `Config` kolona sadrži JSON strukture, često sa više zapisa po jednoj SharePoint stavci.
- Kritične stavke za poslovnu logiku su `DelovodneKnjige`, `Stanja`, `ProcesConfig`, `Permisije`, `EFaktureParametri`, `EfaktureParametri` i `AppLock`.
- Kritične stavke za arhivsku logiku su `ArhivskaKnjigaGrupeKategorija`, `ArhivskaListaKategorije`, `KategorijeDokumentarnogMaterijala`, `RokoviCuvanja`, `RegistracioneJedinice` i `TipRegistracioneJedinice`.
- Kritične stavke za korisnički interfejs su `Translations`, `Users`, `VrsteDokumenta`, `TipoviDokumenta`, `OrganizacioneJedinice` i `Valute`.

## Nepoznato

- Nije potvrđeno koje Power Apps formule direktno čitaju svaku konfiguraciju.
- Nije potvrđeno koji Power Automate flow-ovi koriste svaku konfiguraciju.
- Nije potvrđeno da li je `AppLock` stvarno deo concurrency-safe mehanizma za delovodni broj ili samo administrativni lock.
- Nije potvrđeno da li `Permisije` upravlja stvarnim SharePoint pravima ili samo aplikativnom vidljivošću.

## Preporuka za GitHub strukturu

```text
data-model/
└── app-config/
    ├── README.md
    ├── delovodneknjige.md
    ├── procesconfig.md
    ├── stanja.md
    └── ...
evidence/
└── app-config-json/
    ├── delovodneknjige.json
    ├── procesconfig.json
    └── ...
```
