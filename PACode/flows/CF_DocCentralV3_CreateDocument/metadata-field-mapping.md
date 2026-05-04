# Metadata field mapping: CF_DocCentralV3_CreateDocument

## Purpose

Maps each flow input or derived value to the target SharePoint column for both
the Svi predmeti list (Create item action) and the Dokumenti library (Update file properties action).

**Status: All custom column internal names are UNKNOWN.**
Internal names must be read from each list/library's column settings before building.

To find internal column names:
1. Open the list or library in SharePoint.
2. List Settings (or Library Settings) → Columns → click each column name.
3. Read `Field=<InternalName>` from the URL.

Or use the REST API:
```
GET _api/web/lists/getbytitle('<ListName>')/fields?$filter=Hidden eq false&$select=Title,InternalName,TypeAsString
```

---

## Svi predmeti — field mapping

### Standard / system columns

| Display name | Internal name | Source | Type | Notes |
|---|---|---|---|---|
| Title | `Title` | `varMetadata?['subject']` | Single line of text | Required by SharePoint |
| ID | `ID` | Auto-assigned by SharePoint | Integer | Store as varSviPredmetiItemId |

### Custom columns (internal names UNKNOWN)

| Display name | Internal name | Source | Type | Notes |
|---|---|---|---|---|
| DelovodniBroj | UNKNOWN | `varDelovodniBroj` | Single line of text | Registry number, e.g. `42/2026` |
| Stanje | UNKNOWN | `'Zavedeno'` | Choice | Initial state on creation |
| DocumentType | UNKNOWN | `triggerBody()?['text_documentType']` | Choice or Single line | Document type from input |
| FilingDate | UNKNOWN | `varFilingDate` | Date and Time | Today (generated) or user-selected (reserved) |
| InicijatorEmail | UNKNOWN | `triggerBody()?['text_initiatorEmail']` | Single line of text | |
| PartnerId | UNKNOWN | `varMetadata?['partnerId']` | Number | Nullable — pass null if 0 or empty |
| PartnerNazivSnapshot | UNKNOWN | `varMetadata?['partnerNaziv']` | Single line of text | Partner name at registration time |
| PartnerPIBSnapshot | UNKNOWN | `varMetadata?['partnerPIBSnapshot']` | Single line of text | Partner tax ID at registration time |
| PartnerMestoSnapshot | UNKNOWN | `varMetadata?['partnerMestoSnapshot']` | Single line of text | Partner city at registration time |
| PartnerAdresaSnapshot | UNKNOWN | `varMetadata?['partnerAdresaSnapshot']` | Single line of text | Partner address at registration time |
| Description | UNKNOWN | `varMetadata?['description']` | Multiple lines of text | Optional |
| CorrelationId | UNKNOWN | `varCorrelationId` | Single line of text | For audit log correlation |
| TrenutniOdobravalacEmail | UNKNOWN | (empty on creation) | Single line of text | Set by SendForApproval — not by this flow |
| TrenutnaGrupaOdobravanjaId | UNKNOWN | (empty on creation) | Single line of text | Set by SendForApproval — not by this flow |
| TrenutniKorakOdobravanja | UNKNOWN | (0 on creation) | Number | Set by SendForApproval — not by this flow |
| UkupnoKorakaOdobravanja | UNKNOWN | (0 on creation) | Number | Set by SendForApproval — not by this flow |
| KoraciOdobravanjaJson | UNKNOWN | (empty on creation) | Multiple lines of text | Set by SendForApproval — not by this flow |
| IstorijaOdobravanjaJson | UNKNOWN | (empty on creation) | Multiple lines of text | Set by SendForApproval — not by this flow |
| InicijatorEmail | UNKNOWN | `triggerBody()?['text_initiatorEmail']` | Single line of text | Same as InicijatorEmail — confirm if same column |
| Arhivirano | UNKNOWN | (empty on creation) | Date and Time | Set by ArchiveDocument — not by this flow |
| Arhivirao | UNKNOWN | (empty on creation) | Single line of text | Set by ArchiveDocument — not by this flow |

### Columns intentionally NOT set by this flow

- All approval runtime state fields (TrenutniOdobravalacEmail, TrenutnaGrupaOdobravanjaId, etc.) — set by CF_DocCentralV3_SendForApproval.
- Arhivirano, Arhivirao — set by CF_DocCentralV3_ArchiveDocument.

---

## Dokumenti library — file metadata mapping (main document)

Applied via **SharePoint — Update file properties** after file creation.

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| DelovodniBroj | UNKNOWN | `varDelovodniBroj` | Links file to document |
| SviPredmetiId | UNKNOWN | `varSviPredmetiItemId` | Integer link to Svi predmeti item |
| IsPrilog | UNKNOWN | `false` | Boolean — main file |
| OriginalFileName | UNKNOWN | `varMainFile?['fileName']` | Original name before sanitization |
| SystemFileName | UNKNOWN | `varMainFileSystemName` | Generated unique system name |
| DocumentType | UNKNOWN | `triggerBody()?['text_documentType']` | Mirrors Svi predmeti |
| DocumentStatus | UNKNOWN | `'Zavedeno'` | Mirrors Svi predmeti Stanje |
| CorrelationId | UNKNOWN | `varCorrelationId` | Audit correlation |
| ParentDelovodniBroj | UNKNOWN | (not set for main file) | Only on attachments |
| ParentDocumentId | UNKNOWN | (not set for main file) | Only on attachments |

---

## Dokumenti library — file metadata mapping (attachments)

Applied via **SharePoint — Update file properties** for each attachment inside the loop.

| Display name | Internal name | Value | Notes |
|---|---|---|---|
| DelovodniBroj | UNKNOWN | `varDelovodniBroj` | Same as main document |
| SviPredmetiId | UNKNOWN | `varSviPredmetiItemId` | Same Svi predmeti item ID |
| IsPrilog | UNKNOWN | `true` | Boolean — marks as attachment |
| OriginalFileName | UNKNOWN | `items('Apply_to_each_Attachments')?['fileName']` | Original attachment file name |
| SystemFileName | UNKNOWN | Compose output inside loop | Generated `_PRILOG_` system name |
| DocumentType | UNKNOWN | `triggerBody()?['text_documentType']` | |
| DocumentStatus | UNKNOWN | `'Zavedeno'` | |
| CorrelationId | UNKNOWN | `varCorrelationId` | |
| ParentDelovodniBroj | UNKNOWN | `varDelovodniBroj` | Same as the main document's delovodniBroj |
| ParentDocumentId | UNKNOWN | `varSviPredmetiItemId` | Integer ID of Svi predmeti item |

---

## Choice column values

Confirm that these exact strings exist as Choice options before building:

| Column | Value used by this flow | List/Library |
|---|---|---|
| Stanje (Svi predmeti) | `Zavedeno` | Svi predmeti |
| DocumentStatus (Dokumenti) | `Zavedeno` | Dokumenti library |
| DocumentType | Depends on client configuration (UNKNOWN) | Both |

If the Choice column does not have `Zavedeno` as an option, SharePoint will return 400 Bad Request.
Verify Choice options before building.

---

## Nullable number fields

For fields typed as Number in SharePoint (e.g. PartnerId):

- If the value is null or 0 (not applicable), leave the field unset in the Create item action rather than passing 0.
- Use a Condition before the Create item action to check:
  ```
  equals(variables('varMetadata')?['partnerId'], null)
  ```
  If null → omit the field from the Create item action body.
  If set → include the integer value.

---

## customFields mapping

The `metadataJson.customFields` object contains document-type-specific fields.
The mapping of customFields keys to Svi predmeti internal column names is **UNKNOWN** until:
1. AppConfig.csv is provided and defines all document types.
2. The Svi predmeti list schema is confirmed with all custom type-specific columns.

For v1: customFields are stored as-is in the flow but not mapped to individual columns.
If required, store the entire customFields JSON in a dedicated `CustomFieldsJson` column (UNKNOWN internal name).

---

## Actions to complete before building

1. Open Svi predmeti in SharePoint → List Settings → read every custom column's internal name.
2. Open Dokumenti in SharePoint → Library Settings → read every custom column's internal name.
3. Fill in all UNKNOWN entries in both tables above.
4. Verify Choice options for Stanje and DocumentStatus.
5. Update Create item and Update file properties actions in `implementation-steps.md` with confirmed names.
