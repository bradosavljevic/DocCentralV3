# Data model: SharePoint lista „Svi predmeti”

## Status dokumenta

| Stavka | Vrednost |
|---|---|
| Folder | `data-model` |
| Predloženi fajl | `data-model/svi-predmeti.md` |
| Izvor | SharePoint REST API export liste/kolona u XML formatu |
| Datum exporta iz XML-a | `2026-05-02T08:49:38Z` |
| Site | `https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0` |
| Lista | `Svi predmeti` |
| List GUID | `db7e45e7-2820-43b6-82b8-3cfd58c816d9` |
| Broj kolona u exportu | 126 |
| Vidljive kolone | 65 |
| Skrivene kolone | 61 |

## Svrha liste

Lista `Svi predmeti` je centralna evidencija predmeta/dokumenata u Document Central rešenju. Na osnovu naziva kolona, lista čuva delovodni broj, stanje obrade, vrstu dokumenta, zaduženu osobu, partnera, datume dokumenta, e-dokument status, lokaciju/fasciklu i procesne/metapodatke za tokove obrade.

## Činjenice iz exporta

- Lista ima 126 kolona u dostavljenom exportu.
- Ne postoji kolona sa `Required = true` u dostavljenom exportu.
- Ne postoji kolona sa `EnforceUniqueValues = true` u dostavljenom exportu.
- `DelovodniBroj` nije označen kao unique i nije indeksiran u exportu.
- `EDokument` je Boolean kolona sa podrazumevanom vrednošću `0`.
- `EdocStatus` je Choice kolona sa podrazumevanom vrednošću `New`.
- `LinkDoDokumenta` je Note kolona i ima JSON column formatting koji prikazuje dugme/link „Pregledaj dokument”.

## Sporno / rizici za enterprise verziju

- `DelovodniBroj` nije unique na nivou SharePoint liste. Ako sistem mora garantovati da dva dokumenta nikada ne dobiju isti delovodni broj, ovo mora biti rešeno dodatnom enterprise logikom: atomic/optimistic locking, retry, audit/error log i/ili unique constraint na odgovarajućem polju.
- Veći deo poslovnih kolona je tipa `Text`, uključujući `Partner`, `Stanje`, `TipDokumenta`, `VrstaDokumenta`, `OrganzacionaJedinica`, `Valuta`. To smanjuje referencijalnu kontrolu i otežava validaciju u odnosu na Lookup/Choice model.
- `LinkDoDokumenta` je `Note`, a ne Hyperlink kolona. Ovo može biti namerno zbog Power Automate/SharePoint problema sa URL poljima, ali treba dokumentovati kao tehničku odluku.
- `Treshold` je verovatno typo za `Threshold`; ne menjati bez migracionog plana jer je internal name već deo aplikacije/flow logike.

## Nepoznato

- Iz ovog XML-a nije vidljiv kompletan Power Apps način korišćenja svake kolone.
- Iz ovog XML-a nije vidljiva konfiguracija views, permissions, content types i item-level permissions.
- Iz ovog XML-a nije vidljivo da li Power Automate dodatno garantuje jedinstvenost delovodnog broja.
- Iz ovog XML-a nije vidljivo koje kolone su obavezne u aplikaciji, jer SharePoint `Required` nije uključen ni na jednoj koloni.

## Ključne poslovne kolone

| Display name | Internal name | Static name | Tip | Required | Indexed | Unique | Detalji |
|---|---|---|---|---:|---:|---:|---|
| Title | `Title` | `Title` | Text | Ne | Ne | Ne | - |
| DelovodniBroj | `DelovodniBroj` | `DelovodniBroj` | Text | Ne | Ne | Ne | - |
| Dokument | `Dokument` | `Dokument` | Text | Ne | Ne | Ne | - |
| VrstaDokumenta | `VrstaDokumenta` | `VrstaDokumenta` | Text | Ne | Ne | Ne | - |
| DelovodnaKnjiga | `DelovodnaKnjiga` | `DelovodnaKnjiga` | Text | Ne | Ne | Ne | - |
| Stanje | `Stanje` | `Stanje` | Text | Ne | Ne | Ne | - |
| DatumZavodjenja | `DatumZavodjenja` | `DatumZavodjenja` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| ZaduzenaOsoba | `ZaduzenaOsoba` | `ZaduzenaOsoba` | User | Ne | Ne | Ne | ShowField: ImnName; UserSelectionMode: PeopleOnly |
| IzlazniUlazni | `IzlazniUlazni` | `IzlazniUlazni` | Text | Ne | Ne | Ne | - |
| Partner | `Partner` | `Partner` | Text | Ne | Ne | Ne | - |
| OrganzacionaJedinica | `OrganzacionaJedinica` | `OrganzacionaJedinica` | Text | Ne | Ne | Ne | - |
| Valuta | `Valuta` | `Valuta` | Text | Ne | Ne | Ne | - |
| KlasifikacionaOznaka | `KlasifikacionaOznaka` | `KlasifikacionaOznaka` | Text | Ne | Ne | Ne | - |
| BrojRegistracioneJedinice | `BrojRegistracioneJedinice` | `BrojRegistracioneJedinice` | Text | Ne | Ne | Ne | - |
| DokumentIznos | `DokumentIznos` | `DokumentIznos` | Number | Ne | Ne | Ne | - |
| DokumentaDatum | `DokumentaDatum` | `DokumentaDatum` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| DokumentaDatumIsteka | `DokumentaDatumIsteka` | `DokumentaDatumIsteka` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| DokumentBroj | `DokumentBroj` | `DokumentBroj` | Text | Ne | Ne | Ne | - |
| EdokumentaIPrilozi | `EdokumentaIPrilozi` | `EdokumentaIPrilozi` | Text | Ne | Ne | Ne | - |
| EdocStatus | `EdocStatus` | `EdocStatus` | Choice | Ne | Ne | Ne | Choices: New; Seen; Renotified; Deleted; Approved; Rejected; Cancelled; Storno; SendingInProgress; Unknown; CRF; Reminded; Default: New; Format: Dropdown |
| EdokumentDatumPostavkeNaSef | `EdokumentDatumPostavkeNaSef` | `EdokumentDatumPostavkeNaSef` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| EDokument | `EDokument` | `EDokument` | Boolean | Ne | Ne | Ne | Default: 0; Format: Dropdown |
| Lokacija | `Lokacija` | `Lokacija` | Text | Ne | Ne | Ne | Format: Dropdown |
| Fascikla | `Fascikla` | `Fascikla` | Text | Ne | Ne | Ne | Format: Dropdown |
| AK | `AK` | `AK` | Text | Ne | Ne | Ne | Format: Dropdown |
| AKGodina | `AKGodina` | `AKGodina` | Text | Ne | Ne | Ne | Format: Dropdown |
| Zakljuceno | `Zakljuceno` | `Zakljuceno` | Text | Ne | Ne | Ne | Default: Ne; Format: Dropdown |
| Zaveo | `Zaveo` | `Zaveo` | Text | Ne | Ne | Ne | Format: Dropdown |
| IsProcess | `IsProcess` | `IsProcess` | Number | Ne | Ne | Ne | Format: Dropdown |
| ProcessType | `ProcessType` | `ProcessType` | Text | Ne | Ne | Ne | Format: Dropdown |
| Komentar | `Komentar` | `Komentar` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| Treshold | `Treshold` | `Treshold` | Number | Ne | Ne | Ne | Format: Dropdown |
| Korak | `Korak` | `Korak` | Number | Ne | Ne | Ne | Format: Dropdown |
| ZaveoEmail | `ZaveoEmail` | `ZaveoEmail` | Text | Ne | Ne | Ne | Format: Dropdown |
| AdHocApprove | `AdHocApprove` | `AdHocApprove` | Number | Ne | Ne | Ne | Default: 0; Format: Dropdown |
| Komentar sa SEF-a | `KomentarsaSEF_x002d_a` | `KomentarsaSEF_x002d_a` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| TipDokumenta | `TipDokumenta` | `TipDokumenta` | Text | Ne | Ne | Ne | Format: Dropdown |
| EksterniBrojFakture | `EksterniBrojFakture` | `EksterniBrojFakture` | Text | Ne | Ne | Ne | Format: Dropdown |
| LinkDoDokumenta | `LinkDoDokumenta` | `LinkDoDokumenta` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE; Ima JSON column formatting |
| Komentar na SEF | `KomentarnaSEF` | `KomentarnaSEF` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| BCProvera | `BCProvera` | `BCProvera` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| BCObradjeno | `BCObradjeno` | `BCObradjeno` | Boolean | Ne | Ne | Ne | Default: 0; Format: Dropdown |

## Sve poslovno upisive vidljive kolone

| Display name | Internal name | Static name | Tip | Required | Indexed | Unique | Detalji |
|---|---|---|---|---:|---:|---:|---|
| Title | `Title` | `Title` | Text | Ne | Ne | Ne | - |
| DelovodniBroj | `DelovodniBroj` | `DelovodniBroj` | Text | Ne | Ne | Ne | - |
| Dokument | `Dokument` | `Dokument` | Text | Ne | Ne | Ne | - |
| VrstaDokumenta | `VrstaDokumenta` | `VrstaDokumenta` | Text | Ne | Ne | Ne | - |
| DelovodnaKnjiga | `DelovodnaKnjiga` | `DelovodnaKnjiga` | Text | Ne | Ne | Ne | - |
| Stanje | `Stanje` | `Stanje` | Text | Ne | Ne | Ne | - |
| DatumZavodjenja | `DatumZavodjenja` | `DatumZavodjenja` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| ZaduzenaOsoba | `ZaduzenaOsoba` | `ZaduzenaOsoba` | User | Ne | Ne | Ne | ShowField: ImnName; UserSelectionMode: PeopleOnly |
| IzlazniUlazni | `IzlazniUlazni` | `IzlazniUlazni` | Text | Ne | Ne | Ne | - |
| Partner | `Partner` | `Partner` | Text | Ne | Ne | Ne | - |
| OrganzacionaJedinica | `OrganzacionaJedinica` | `OrganzacionaJedinica` | Text | Ne | Ne | Ne | - |
| Valuta | `Valuta` | `Valuta` | Text | Ne | Ne | Ne | - |
| KlasifikacionaOznaka | `KlasifikacionaOznaka` | `KlasifikacionaOznaka` | Text | Ne | Ne | Ne | - |
| BrojRegistracioneJedinice | `BrojRegistracioneJedinice` | `BrojRegistracioneJedinice` | Text | Ne | Ne | Ne | - |
| DokumentIznos | `DokumentIznos` | `DokumentIznos` | Number | Ne | Ne | Ne | - |
| DokumentaDatum | `DokumentaDatum` | `DokumentaDatum` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| DokumentaDatumIsteka | `DokumentaDatumIsteka` | `DokumentaDatumIsteka` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| DokumentBroj | `DokumentBroj` | `DokumentBroj` | Text | Ne | Ne | Ne | - |
| EdokumentaIPrilozi | `EdokumentaIPrilozi` | `EdokumentaIPrilozi` | Text | Ne | Ne | Ne | - |
| EdocStatus | `EdocStatus` | `EdocStatus` | Choice | Ne | Ne | Ne | Choices: New; Seen; Renotified; Deleted; Approved; Rejected; Cancelled; Storno; SendingInProgress; Unknown; CRF; Reminded; Default: New; Format: Dropdown |
| EdokumentDatumPostavkeNaSef | `EdokumentDatumPostavkeNaSef` | `EdokumentDatumPostavkeNaSef` | DateTime | Ne | Ne | Ne | Format: DateOnly |
| EDokument | `EDokument` | `EDokument` | Boolean | Ne | Ne | Ne | Default: 0; Format: Dropdown |
| Lokacija | `Lokacija` | `Lokacija` | Text | Ne | Ne | Ne | Format: Dropdown |
| Fascikla | `Fascikla` | `Fascikla` | Text | Ne | Ne | Ne | Format: Dropdown |
| AK | `AK` | `AK` | Text | Ne | Ne | Ne | Format: Dropdown |
| AKGodina | `AKGodina` | `AKGodina` | Text | Ne | Ne | Ne | Format: Dropdown |
| Zakljuceno | `Zakljuceno` | `Zakljuceno` | Text | Ne | Ne | Ne | Default: Ne; Format: Dropdown |
| Zaveo | `Zaveo` | `Zaveo` | Text | Ne | Ne | Ne | Format: Dropdown |
| IsProcess | `IsProcess` | `IsProcess` | Number | Ne | Ne | Ne | Format: Dropdown |
| ProcessType | `ProcessType` | `ProcessType` | Text | Ne | Ne | Ne | Format: Dropdown |
| Komentar | `Komentar` | `Komentar` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| Treshold | `Treshold` | `Treshold` | Number | Ne | Ne | Ne | Format: Dropdown |
| Korak | `Korak` | `Korak` | Number | Ne | Ne | Ne | Format: Dropdown |
| ZaveoEmail | `ZaveoEmail` | `ZaveoEmail` | Text | Ne | Ne | Ne | Format: Dropdown |
| AdHocApprove | `AdHocApprove` | `AdHocApprove` | Number | Ne | Ne | Ne | Default: 0; Format: Dropdown |
| Komentar sa SEF-a | `KomentarsaSEF_x002d_a` | `KomentarsaSEF_x002d_a` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| TipDokumenta | `TipDokumenta` | `TipDokumenta` | Text | Ne | Ne | Ne | Format: Dropdown |
| EksterniBrojFakture | `EksterniBrojFakture` | `EksterniBrojFakture` | Text | Ne | Ne | Ne | Format: Dropdown |
| LinkDoDokumenta | `LinkDoDokumenta` | `LinkDoDokumenta` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE; Ima JSON column formatting |
| Komentar na SEF | `KomentarnaSEF` | `KomentarnaSEF` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| BCProvera | `BCProvera` | `BCProvera` | Note | Ne | Ne | Ne | Format: Dropdown; RichText: FALSE; AppendOnly: FALSE |
| BCObradjeno | `BCObradjeno` | `BCObradjeno` | Boolean | Ne | Ne | Ne | Default: 0; Format: Dropdown |

## Vidljive sistemske / read-only kolone

| Display name | Internal name | Static name | Tip | Required | Indexed | Unique | Detalji |
|---|---|---|---|---:|---:|---:|---|
| Color Tag | `_ColorTag` | `_ColorTag` | Text | Ne | Ne | Ne | - |
| Compliance Asset Id | `ComplianceAssetId` | `ComplianceAssetId` | Text | Ne | Ne | Ne | - |
| Created | `Created` | `Created` | DateTime | Ne | Ne | Ne | - |
| Modified | `Modified` | `Modified` | DateTime | Ne | Ne | Ne | - |
| ID | `ID` | `ID` | Counter | Ne | Ne | Ne | - |
| Content Type | `ContentType` | `ContentType` | Computed | Ne | Ne | Ne | - |
| Created By | `Author` | `Author` | User | Ne | Ne | Ne | - |
| Modified By | `Editor` | `Editor` | User | Ne | Ne | Ne | - |
| Version | `_UIVersionString` | `_UIVersionString` | Text | Ne | Ne | Ne | - |
| Attachments | `Attachments` | `Attachments` | Attachments | Ne | Ne | Ne | - |
| Edit | `Edit` | `Edit` | Computed | Ne | Ne | Ne | - |
| Title | `LinkTitleNoMenu` | `LinkTitleNoMenu` | Computed | Ne | Ne | Ne | - |
| Title | `LinkTitle` | `LinkTitle` | Computed | Ne | Ne | Ne | - |
| Type | `DocIcon` | `DocIcon` | Computed | Ne | Ne | Ne | - |
| Item Child Count | `ItemChildCount` | `ItemChildCount` | Lookup | Ne | Ne | Ne | ShowField: ItemChildCount |
| Folder Child Count | `FolderChildCount` | `FolderChildCount` | Lookup | Ne | Ne | Ne | ShowField: FolderChildCount |
| Label setting | `_ComplianceFlags` | `_ComplianceFlags` | Lookup | Ne | Ne | Ne | ShowField: ComplianceFlags |
| Retention label | `_ComplianceTag` | `_ComplianceTag` | Lookup | Ne | Ne | Ne | ShowField: ComplianceTag |
| Retention label Applied | `_ComplianceTagWrittenTime` | `_ComplianceTagWrittenTime` | Lookup | Ne | Ne | Ne | ShowField: ComplianceTagWrittenTime |
| Label applied by | `_ComplianceTagUserId` | `_ComplianceTagUserId` | Lookup | Ne | Ne | Ne | ShowField: ComplianceTagUserId |
| Item is a Record | `_IsRecord` | `_IsRecord` | Computed | Ne | Ne | Ne | - |
| App Created By | `AppAuthor` | `AppAuthor` | Lookup | Ne | Ne | Ne | ShowField: Title |
| App Modified By | `AppEditor` | `AppEditor` | Lookup | Ne | Ne | Ne | ShowField: Title |

## Skrivene kolone iz exporta

| Display name | Internal name | Tip | Read-only | Detalji |
|---|---|---|---:|---|
| Content Type ID | `ContentTypeId` | ContentTypeId | Da | - |
| Approver Comments | `_ModerationComments` | Note | Da | - |
| File Type | `File_x0020_Type` | Text | Da | - |
| Color | `_ColorHex` | Text | Da | - |
| Emoji | `_Emoji` | Text | Da | - |
| Has Copy Destinations | `_HasCopyDestinations` | Boolean | Da | - |
| Copy Source | `_CopySource` | Text | Da | - |
| owshiddenversion | `owshiddenversion` | Integer | Da | - |
| Workflow Version | `WorkflowVersion` | Integer | Da | - |
| UI Version | `_UIVersion` | Integer | Da | - |
| Approval Status | `_ModerationStatus` | ModStat | Da | Choices: 0;#Approved; 1;#Rejected; 2;#Pending; 3;#Draft; 4;#Scheduled; Default: 0 |
| Title | `LinkTitle2` | Computed | Da | - |
| Select | `SelectTitle` | Computed | Da | - |
| Instance ID | `InstanceID` | Integer | Da | - |
| Order | `Order` | Number | Ne | - |
| GUID | `GUID` | Guid | Da | - |
| Workflow Instance ID | `WorkflowInstanceID` | Guid | Da | - |
| URL Path | `FileRef` | Lookup | Da | - |
| Path | `FileDirRef` | Lookup | Da | - |
| Modified | `Last_x0020_Modified` | Lookup | Da | - |
| Created | `Created_x0020_Date` | Lookup | Da | - |
| Item Type | `FSObjType` | Lookup | Da | - |
| Sort Type | `SortBehavior` | Lookup | Da | - |
| Effective Permissions Mask | `PermMask` | Computed | Da | - |
| Principal Count | `PrincipalCount` | Computed | Da | - |
| Name | `FileLeafRef` | File | Ne | - |
| Unique Id | `UniqueId` | Lookup | Da | - |
| Document Parent Identifier | `ParentUniqueId` | Lookup | Da | - |
| Client Id | `SyncClientId` | Lookup | Da | - |
| ProgId | `ProgId` | Lookup | Da | - |
| ScopeId | `ScopeId` | Lookup | Da | - |
| HTML File Type | `HTML_x0020_File_x0020_Type` | Computed | Da | - |
| Edit Menu Table Start | `_EditMenuTableStart` | Computed | Da | - |
| Edit Menu Table Start | `_EditMenuTableStart2` | Computed | Da | - |
| Edit Menu Table End | `_EditMenuTableEnd` | Computed | Da | - |
| Name | `LinkFilenameNoMenu` | Computed | Da | - |
| Name | `LinkFilename` | Computed | Da | - |
| Name | `LinkFilename2` | Computed | Da | - |
| Server Relative URL | `ServerUrl` | Computed | Da | - |
| Encoded Absolute URL | `EncodedAbsUrl` | Computed | Da | - |
| File Name | `BaseName` | Computed | Da | - |
| Property Bag | `MetaInfo` | Lookup | Ne | - |
| Level | `_Level` | Integer | Da | - |
| Is Current Version | `_IsCurrentVersion` | Boolean | Da | - |
| Restricted | `Restricted` | Lookup | Da | - |
| Originator Id | `OriginatorId` | Lookup | Da | - |
| NoExecute | `NoExecute` | Lookup | Da | - |
| Content Version | `ContentVersion` | Lookup | Da | - |
| Access Policy | `AccessPolicy` | Lookup | Da | - |
| VirusStatus | `_VirusStatus` | Lookup | Da | - |
| VirusVendorID | `_VirusVendorID` | Lookup | Da | - |
| VirusInfo | `_VirusInfo` | Lookup | Da | - |
| RansomwareAnomalyMetaInfo | `_RansomwareAnomalyMetaInfo` | Lookup | Da | - |
| Draft Owner Id | `_DraftOwnerId` | Lookup | Da | - |
| Main Link settings | `MainLinkSettings` | Computed | Da | - |
| Total Size | `SMTotalSize` | Lookup | Da | - |
| Last Modified Date | `SMLastModifiedDate` | Lookup | Da | - |
| Total File Stream Size | `SMTotalFileStreamSize` | Lookup | Da | - |
| Total File Count | `SMTotalFileCount` | Lookup | Da | - |
| Comment settings | `_CommentFlags` | Lookup | Da | - |
| Comment count | `_CommentCount` | Lookup | Da | - |

## Posebna napomena: `EdocStatus`

`EdocStatus` je Choice kolona sa sledećim vrednostima:
- `New`
- `Seen`
- `Renotified`
- `Deleted`
- `Approved`
- `Rejected`
- `Cancelled`
- `Storno`
- `SendingInProgress`
- `Unknown`
- `CRF`
- `Reminded`

## Posebna napomena: `LinkDoDokumenta`

Kolona `LinkDoDokumenta` je tipa `Note`, ali ima SharePoint JSON column formatting. Formatiranje pravi vizuelno dugme koje otvara vrednost kolone kao link u novom tabu. Ovo treba tretirati kao UI-formatirano tekstualno URL polje, ne kao SharePoint Hyperlink field.

## Preporuke za novu verziju data modela

1. Uvesti concurrency-safe mehanizam za generisanje i rezervaciju `DelovodniBroj`.
2. Razmotriti indeksiranje često filtriranih kolona: `DelovodniBroj`, `Stanje`, `TipDokumenta`, `EDokument`, `DatumZavodjenja`, `Partner`, `ZaduzenaOsoba`.
3. Razmotriti prebacivanje poslovnih klasifikacija iz slobodnog teksta u Choice/Lookup tabele: `Stanje`, `TipDokumenta`, `VrstaDokumenta`, `OrganzacionaJedinica`, `Valuta`, `Partner`.
4. Uvesti dokumentovanu audit/error log listu za neuspele upise, retry pokušaje i konflikt delovodnog broja.
5. Validacije koje su sada verovatno u Canvas aplikaciji/flow-ovima treba eksplicitno dokumentovati, jer SharePoint lista nema required kolone u exportu.

## REST link za ponovno čitanje kolona

```text
https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0/_api/web/lists/getbytitle('Svi%20predmeti')/fields?$select=Title,InternalName,StaticName,TypeAsString,Required,Hidden,ReadOnlyField,Indexed,EnforceUniqueValues,DefaultValue,Description,Group,SchemaXml
```

## GitHub komande

Komande za dodavanje ovog fajla u repo:
```bash
mkdir -p data-model
# kopiraj fajl u: data-model/svi-predmeti.md
git add data-model/svi-predmeti.md
git commit -m "docs: add Svi predmeti data model"
git push
```
