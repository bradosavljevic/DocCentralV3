# Reserved number field mapping: CF_DocCentralV3_UseReservedNumber

## Purpose

Documents which columns this flow reads from RezervisaniBrojevi and what App Config entries
it reads for year validation. Used when configuring SharePoint actions in Power Automate.

**Status: All internal column names for RezervisaniBrojevi are UNKNOWN.**
Confirm by opening the list in SharePoint → List Settings → Columns → click each column name
and read the `Field=<InternalName>` URL parameter.

---

## RezervisaniBrojevi list

| What the flow reads | Display name | Internal column name | SP type | Notes |
|---|---|---|---|---|
| Registry number string | DelovodniBroj | UNKNOWN | Single line of text | Used as the output delovodniBroj and for duplicate check in Svi predmeti |
| Year the number is reserved for | UNKNOWN | UNKNOWN | Number or Choice | Must be confirmed from list schema. Flow reads this to validate against requestedYear. |
| Suggested filing date | UNKNOWN | UNKNOWN | Date and Time | Optional. Returned to caller as filingDateSuggestion. May not exist — mark UNKNOWN. |
| SharePoint item ID | ID | ID | Integer (system) | Passed in by caller as reservedNumberId. Used in Get item and Delete item. |
| ETag | @odata.etag | @odata.etag | System field | Captured for future soft-lock use. Not used for PATCH in this flow version. |

### How to find internal column names

1. Open RezervisaniBrojevi list in SharePoint.
2. Settings → List Settings → Columns section.
3. Click each column name. Read `Field=<InternalName>` from the URL.

Or via REST API (run while logged in as service account):
```
GET _api/web/lists/getbytitle('RezervisaniBroji')/fields?$filter=Hidden eq false&$select=Title,InternalName,TypeAsString
```
Replace the list name with the actual display name of the RezervisaniBrojevi list.

---

## App Config entries read by this flow

This flow reads the active registry year from App Config for the YEAR_NOT_ACTIVE validation check.

| What the flow reads | App Config key | Internal column name | Notes |
|---|---|---|---|
| Active year | UNKNOWN | UNKNOWN | Same entry read by CF_DocCentralV3_GenerateRegistryNumber |

The filter query and column name are UNKNOWN until AppConfig.csv is provided.
See `PACode/flows/CF_DocCentralV3_GenerateRegistryNumber/appconfig-field-mapping.md` for the
pattern — once that file is updated with confirmed values, use the same filter here.

---

## Svi predmeti list — duplicate check column

This flow queries Svi predmeti to check whether the delovodniBroj has already been used.

| What the flow checks | Display name | Internal column name | Notes |
|---|---|---|---|
| Registry number on existing documents | DelovodniBroj | UNKNOWN | Filter: `DelovodniBroj eq '<varDelovodniBroj>'` — replace with actual internal name |

---

## Expressions to update once column names are confirmed

Update the following expressions in `implementation-steps.md` Step 5a when internal names are known:

**varDelovodniBroj assignment:**
```
outputs('Get_item')?['body']?['UNKNOWN_DelovodniBrojColumn']
```

**varReservedYear assignment:**
```
int(outputs('Get_item')?['body']?['UNKNOWN_YearColumn'])
```

**filingDateSuggestion in Respond action:**
```
outputs('Get_item')?['body']?['UNKNOWN_FilingDateColumn']
```

**Svi predmeti filter query:**
```
UNKNOWN_DelovodniBrojInternalName eq '@{variables('varDelovodniBroj')}'
```

**App Config filter query (active year row):**
Same as CF_DocCentralV3_GenerateRegistryNumber — UNKNOWN until AppConfig.csv is available.

---

## Validation dependency summary

| Validation | Lists read | Column(s) read |
|---|---|---|
| Number exists | RezervisaniBrojevi | ID (by item ID lookup) |
| Year match | RezervisaniBrojevi | UNKNOWN year column |
| Active year check | App Config | UNKNOWN key + year column |
| Already used check | Svi predmeti | UNKNOWN DelovodniBroj column |

All four validations must pass for the flow to return success.

---

## Deletion — not performed by this flow

The Delete item action on RezervisaniBrojevi is NOT in this flow.
It belongs to CF_DocCentralV3_CreateDocument, called after Svi predmeti item creation succeeds.

The delete target:
- List: `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')`
- ID: the `reservedNumberId` passed to this flow

This separation ensures reserved numbers are never consumed if document creation fails.

---

## Actions to complete before building

1. Open RezervisaniBrojevi in SharePoint, read internal column names for:
   - DelovodniBroj column
   - Year column (display name UNKNOWN)
   - Filing date column (display name UNKNOWN, may not exist)
2. Confirm App Config active year key (shared with GenerateRegistryNumber).
3. Confirm Svi predmeti internal column name for DelovodniBroj.
4. Update all UNKNOWN entries in this file.
5. Update expressions in `implementation-steps.md`.
