# Data model: SharePoint lista „Partneri”

## Status dokumenta

| Stavka | Vrednost |
|---|---|
| Folder | `data-model` |
| Predloženi fajl | `data-model/partneri.md` |
| Izvor | SharePoint REST API export liste/kolona u XML formatu |
| Datum exporta iz XML-a | `2026-05-02T08:50:40Z` |
| Site | `https://goprobeograd.sharepoint.com/sites/DocumentCentralv6.0` |
| Lista | `Partneri` |
| List GUID | `a9c6ba91-4b8c-4d2f-b62a-f0b4c809ff76` |
| Broj kolona u exportu | 94 |
| Vidljive kolone | 33 |
| Skrivene kolone | 61 |

## Svrha liste

Lista `Partneri` je šifarnik poslovnih partnera/kontakata u Document Central rešenju. Na osnovu kolona iz exporta, lista čuva naziv partnera, adresne podatke, tip kontakta, identifikacione brojeve i Business Central identifikator.

## Činjenice iz exporta

- Lista ima 94 kolona u dostavljenom exportu.
- Poslovno upisive vidljive kolone u exportu su: `Title`, `Adresa`, `Grad`, `TipKontakta`, `PIB`, `JMBG`, `VrstaKontakta`, `MaticniBroj`, `BCID`, `Klasifikacija`, `Attachments`.
- Kolona `Title` je jedina poslovna kolona označena kao `Required = true` u exportu.
- Ne postoji kolona sa `EnforceUniqueValues = true` u dostavljenom exportu.
- Ne postoji indeksirana poslovna kolona u dostavljenom exportu.
- `TipKontakta` je `Choice` kolona, ali u exportu nema definisanih choice vrednosti (`<CHOICES />`).
- `PIB`, `JMBG`, `MaticniBroj`, `BCID` i `Klasifikacija` su `Text` kolone.

## Sporno / rizici za enterprise verziju

- `PIB` nije označen kao unique i nije indeksiran. Ako se partner proverava po PIB-u, preporuka je da `PIB` bude indeksiran, a jedinstvenost mora biti rešena kontrolisano u skladu sa poslovnim pravilima.
- `BCID` nije označen kao unique i nije indeksiran. Ako je `BCID` tehnički identifikator iz Business Central/SAP integracije, treba definisati da li mora biti jedinstven.
- `TipKontakta` je `Choice` kolona bez vrednosti u exportu. To može značiti da vrednosti nisu definisane, da su uklonjene ili da se vrednost popunjava drugačijom logikom.
- `VrstaKontakta` je `Text`, dok je `TipKontakta` `Choice`. Potrebno je potvrditi razliku između ova dva polja, jer naziv sugeriše sličnu poslovnu namenu.
- Za automatsko ažuriranje partnera iz SAP-a/BC-a treba dokumentovati primarni ključ poređenja: `PIB`, `BCID`, `MaticniBroj` ili kombinacija polja.

## Nepoznato

- Iz ovog XML-a nije vidljivo kako Power Apps koristi listu `Partneri`.
- Iz ovog XML-a nije vidljivo da li postoje views, item-level permissions, Power Automate validacije ili integraciona pravila.
- Iz ovog XML-a nije vidljivo koje kolone se popunjavaju iz SAP-a, Business Central-a ili ručnim unosom.
- Iz ovog XML-a nije vidljivo da li se `Title` koristi kao poslovni naziv partnera, tehnički ID ili kombinovana vrednost.

## Ključne poslovne kolone

| Display name | Internal name | Static name | Tip | Required | Indexed | Unique | Detalji |
|---|---|---|---|---|---|---|---|
| Title | `Title` | `Title` | Text | Da | Ne | Ne | - |
| Adresa | `Adresa` | `Adresa` | Text | Ne | Ne | Ne | - |
| Grad | `Grad` | `Grad` | Text | Ne | Ne | Ne | - |
| TipKontakta | `TipKontakta` | `TipKontakta` | Choice | Ne | Ne | Ne | - |
| PIB | `PIB` | `PIB` | Text | Ne | Ne | Ne | - |
| JMBG | `JMBG` | `JMBG` | Text | Ne | Ne | Ne | - |
| VrstaKontakta | `VrstaKontakta` | `VrstaKontakta` | Text | Ne | Ne | Ne | MaxLength: 255 |
| MaticniBroj | `MaticniBroj` | `MaticniBroj` | Text | Ne | Ne | Ne | MaxLength: 255 |
| BCID | `BCID` | `BCID` | Text | Ne | Ne | Ne | MaxLength: 255 |
| Klasifikacija | `Klasifikacija` | `Klasifikacija` | Text | Ne | Ne | Ne | MaxLength: 255 |
| Attachments | `Attachments` | `Attachments` | Attachments | Ne | Ne | Ne | - |

## Vidljive sistemske / read-only kolone

| Display name | Internal name | Static name | Tip | Required | Indexed | Unique | Detalji |
|---|---|---|---|---|---|---|---|
| Content Type | `ContentType` | `ContentType` | Computed | Ne | Ne | Ne | - |
| Color Tag | `_ColorTag` | `_ColorTag` | Text | Ne | Ne | Ne | - |
| Compliance Asset Id | `ComplianceAssetId` | `ComplianceAssetId` | Text | Ne | Ne | Ne | - |
| ID | `ID` | `ID` | Counter | Ne | Ne | Ne | - |
| Modified | `Modified` | `Modified` | DateTime | Ne | Ne | Ne | - |
| Created | `Created` | `Created` | DateTime | Ne | Ne | Ne | - |
| Created By | `Author` | `Author` | User | Ne | Ne | Ne | - |
| Modified By | `Editor` | `Editor` | User | Ne | Ne | Ne | - |
| Version | `_UIVersionString` | `_UIVersionString` | Text | Ne | Ne | Ne | - |
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
| URL Path | `FileRef` | Lookup | Da | ShowField: FullUrl |
| Path | `FileDirRef` | Lookup | Da | ShowField: DirName |
| Modified | `Last_x0020_Modified` | Lookup | Da | Format: TRUE; ShowField: TimeLastModified |
| Created | `Created_x0020_Date` | Lookup | Da | Format: TRUE; ShowField: TimeCreated |
| Item Type | `FSObjType` | Lookup | Da | ShowField: FSType |
| Sort Type | `SortBehavior` | Lookup | Da | ShowField: SortBehavior |
| Effective Permissions Mask | `PermMask` | Computed | Da | - |
| Principal Count | `PrincipalCount` | Computed | Da | - |
| Name | `FileLeafRef` | File | Ne | ShowField: LeafName |
| Unique Id | `UniqueId` | Lookup | Da | ShowField: UniqueId |
| Document Parent Identifier | `ParentUniqueId` | Lookup | Da | ShowField: ParentUniqueId |
| Client Id | `SyncClientId` | Lookup | Da | ShowField: SyncClientId |
| ProgId | `ProgId` | Lookup | Da | ShowField: ProgId |
| ScopeId | `ScopeId` | Lookup | Da | ShowField: ScopeId |
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
| Property Bag | `MetaInfo` | Lookup | Ne | ShowField: MetaInfo |
| Level | `_Level` | Integer | Da | - |
| Is Current Version | `_IsCurrentVersion` | Boolean | Da | - |
| Restricted | `Restricted` | Lookup | Da | ShowField: Restricted |
| Originator Id | `OriginatorId` | Lookup | Da | ShowField: OriginatorId |
| NoExecute | `NoExecute` | Lookup | Da | ShowField: DocFlags |
| Content Version | `ContentVersion` | Lookup | Da | ShowField: ContentVersion |
| Access Policy | `AccessPolicy` | Lookup | Da | ShowField: AccessPolicy |
| VirusStatus | `_VirusStatus` | Lookup | Da | ShowField: VirusStatus |
| VirusVendorID | `_VirusVendorID` | Lookup | Da | ShowField: VirusVendorID |
| VirusInfo | `_VirusInfo` | Lookup | Da | ShowField: VirusInfo |
| RansomwareAnomalyMetaInfo | `_RansomwareAnomalyMetaInfo` | Lookup | Da | ShowField: RansomwareAnomalyMetaInfo |
| Draft Owner Id | `_DraftOwnerId` | Lookup | Da | ShowField: DraftOwnerId |
| Main Link settings | `MainLinkSettings` | Computed | Da | - |
| Total Size | `SMTotalSize` | Lookup | Da | ShowField: SMTotalSize |
| Last Modified Date | `SMLastModifiedDate` | Lookup | Da | ShowField: SMLastModifiedDate |
| Total File Stream Size | `SMTotalFileStreamSize` | Lookup | Da | ShowField: SMTotalFileStreamSize |
| Total File Count | `SMTotalFileCount` | Lookup | Da | ShowField: SMTotalFileCount |
| Comment settings | `_CommentFlags` | Lookup | Da | ShowField: CommentFlags |
| Comment count | `_CommentCount` | Lookup | Da | ShowField: CommentCount |

## Preporuke za dokumentaciju nove verzije

- Definisati jedinstveni identifikator partnera: `PIB`, `BCID`, `MaticniBroj` ili kombinacija polja.
- Ako se partneri sinhronizuju iz eksternog sistema, dokumentovati pravilo za update postojećeg partnera: poređenje po primarnom ključu, zatim ažuriranje polja `Title`, `Adresa`, `Grad`, `PIB`, `MaticniBroj`, `BCID`, `Klasifikacija`.
- Dodati audit log za promene partnera koje dolaze iz integracije: prethodna vrednost, nova vrednost, izvor, vreme izmene i flow run ID.
- Indeksirati kolone koje se koriste u `Get items` filterima, posebno `PIB`, `BCID` i/ili `MaticniBroj`, nakon potvrde stvarnog upita u flow-u.
- Ne menjati internal names postojećih kolona bez migracionog plana, jer ih Power Apps i Power Automate verovatno već koriste.
