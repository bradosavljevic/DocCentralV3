# Power Automate build notes: CF_DocCentralV3_CreateDocument

## Purpose of this document

Practical notes and gotchas for building this flow in the Power Automate maker portal.
This is the most complex flow in the solution. Read all sibling documents before starting.

---

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_CreateDocument |
| Solution | DocCentralV3 |
| Trigger | Power Apps (V2) — called directly from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint, CR_DocCentralV3_Office365Users |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog, EV_DocCentralV3_docDokumenti, EV_DocCentralV3_lstRezervisaniBrojevi |

---

## Power Apps V2 trigger — input types

| Input name | Type | Internal reference |
|---|---|---|
| documentType | Text | `triggerBody()?['text_documentType']` |
| useReservedNumber | Boolean | `triggerBody()?['bool_useReservedNumber']` |
| reservedNumberId | Number | `triggerBody()?['decimal_reservedNumberId']` |
| filingDate | Text | `triggerBody()?['text_filingDate']` |
| initiatorEmail | Text | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `triggerBody()?['text_correlationId']` |
| metadataJson | Text | `triggerBody()?['text_metadataJson']` |
| mainFileJson | Text | `triggerBody()?['text_mainFileJson']` |
| attachmentsJson | Text | `triggerBody()?['text_attachmentsJson']` |

Always verify actual internal names in the expression editor after adding inputs.

---

## JSON input parsing — null-safe patterns

Parse all JSON string inputs immediately in Initialize variable actions:

```
varMetadata = json(triggerBody()?['text_metadataJson'])
varMainFile = json(triggerBody()?['text_mainFileJson'])
varAttachments = json(if(empty(triggerBody()?['text_attachmentsJson']), '[]', triggerBody()?['text_attachmentsJson']))
```

If the Canvas App sends `null` instead of `"[]"` for attachmentsJson, `json(null)` will fail.
The `if(empty(...), '[]', ...)` guard prevents this.

If metadataJson or mainFileJson can be null from the Canvas App, add similar guards:
```
json(if(empty(triggerBody()?['text_metadataJson']), '{}', triggerBody()?['text_metadataJson']))
```

---

## Accessing nested JSON values after parsing

After `json()` parsing, access nested properties using `?['key']` syntax:

```
variables('varMetadata')?['subject']
variables('varMetadata')?['partnerId']
variables('varMainFile')?['fileName']
variables('varMainFile')?['fileContentBase64']
items('Apply_to_each_Attachments')?['fileName']
items('Apply_to_each_Attachments')?['fileContentBase64']
```

If a key is absent from the JSON, `?['key']` returns null — safer than `['key']` which throws.

---

## File content — base64ToBinary

The SharePoint Create file action requires binary file content.
**Do not use** `decodeBase64()` — it produces a string.
**Use** `base64ToBinary()`:

```
base64ToBinary(variables('varMainFile')?['fileContentBase64'])
```

For attachments inside the loop:
```
base64ToBinary(items('Apply_to_each_Attachments')?['fileContentBase64'])
```

---

## Folder path for Create file — library root

The Folder Path field in SharePoint Create file must point to the Dokumenti library root.
The EV `EV_DocCentralV3_docDokumenti` should contain the server-relative URL of the library.

Example value: `/sites/DocCentralV3/Dokumenti`

In the action:
- Site Address: `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')`
- Folder Path: `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')`

Do not append a trailing `/` to the folder path — Power Automate handles this.
Test once manually before building the full flow to confirm the path format works for the site.

---

## Accessing file item ID after Create file

After SharePoint Create file:
```
outputs('Create_MainFile')?['body']?['ItemId']
```

This is an integer. Wrap in `int()` when storing:
```
int(outputs('Create_MainFile')?['body']?['ItemId'])
```

If the action is renamed (e.g. `Create_AttachmentFile`), use:
```
int(outputs('Create_AttachmentFile')?['body']?['ItemId'])
```

---

## Accumulating file IDs across the attachments loop

varCreatedFileIds starts as an empty array `[]`.
After creating the main file, append the ID:
```
union(variables('varCreatedFileIds'), array(variables('varMainFileId')))
```

Inside the attachment loop, after each Create file action:
```
union(variables('varCreatedFileIds'), array(int(outputs('Create_AttachmentFile')?['body']?['ItemId'])))
```

Power Automate Set variable with this union expression overwrites varCreatedFileIds with the extended array.

When passing varCreatedFileIds to CF_DocCentralV3_AssignPermissions (Text input), serialize:
```
string(variables('varCreatedFileIds'))
```
This produces `[101,102,103]` — valid JSON array string that AssignPermissions will parse with `json()`.

---

## guid() in file names inside Apply to each

`guid()` is evaluated at expression execution time. Each time the loop iterates,
a fresh GUID is generated. You do not need to pre-generate GUIDs — use `guid()` directly
inside the Compose action that builds the attachment system name within each iteration.

---

## Terminate Succeeded vs Terminate Failed — critical distinction

- **Terminate Succeeded**: exits the Try scope cleanly. Catch scope does NOT run.
  Use for validation failures — the flow has already responded; no unhandled exception occurred.
- **Terminate Failed**: exits the Try scope as a failure. Catch scope DOES run.
  Use only for unexpected infrastructure failures (SharePoint write errors) where the Catch scope must log and respond.

**Never use Terminate Failed after a Respond action** — the Catch scope will attempt a second Respond, causing the flow to error.

Pattern for each validation failure branch:
```
[Respond to PowerApp — failure response]
[Terminate — Status: Succeeded]
```

Pattern for infrastructure failures (e.g. Create item fails):
```
[Do NOT Respond here]
[Terminate — Status: Failed]
→ Catch scope handles Respond
```

---

## Try scope with many Terminate actions

The Try scope will have many exit points (one per validation check, one per number acquisition failure).
This is expected. Power Automate supports multiple Terminate Succeeded actions within a Try scope.

Keep each validation group visually distinct:
- Name each Condition: `Check_DocumentType`, `Check_MainFile_FileName`, etc.
- Name each Terminate: `Exit_VALIDATION_FAILED_DocumentType`, etc.

This makes the flow readable and the run history traceable.

---

## Compensation delete in Catch scope

The Catch scope must check whether Svi predmeti item was partially created before responding.

```
[Condition: variables('varSviPredmetiItemId') > 0]
  Yes:
    [Delete item — Svi predmeti — varSviPredmetiItemId]
    [Condition: Delete succeeded?]
      Yes: [Log compensation success]
      No:  [Log Critical — orphan item, manual intervention required]
    [Condition: variables('varMainFileId') > 0]
      Yes: [Delete item — Dokumenti — varMainFileId]
[Log CreateDocument / Error / Failed / CREATE_DOCUMENT_FAILED]
[Respond failure]
```

The compensation delete in Catch scope must use Configure run after = `is successful` on the Delete item action
(default — don't change it). If the Delete item itself fails, the nested Condition catches it by checking
`outputs('Delete_item')?['statusCode']`.

---

## Rename all actions immediately

Rename every action as you add it. With 30+ actions in this flow, default names like
"Run a Child Flow", "Run a Child Flow 2", "Condition", "Condition 3" make the flow impossible
to maintain and expressions impossible to read.

Suggested naming convention:
- Initialization: `Init_varCorrelationId`, `Init_varMetadata`, etc.
- Validation conditions: `Check_DocumentType_Empty`, `Check_MainFile_Content_Empty`
- Child flow calls: `Call_LogEvent_Started`, `Call_GenerateRegistryNumber`, `Call_UseReservedNumber`, `Call_AssignPermissions`
- SharePoint actions: `Get_AppConfig_DocumentTypes`, `Get_AppConfig_ActiveYear`, `Create_SviPredmeti_Item`, `Create_MainFile`, `Update_MainFile_Metadata`
- Loops: `Apply_to_each_Attachments`
- Inside loop: `Build_AttachmentSystemName`, `Create_AttachmentFile`, `Update_AttachmentFile_Metadata`
- Cleanup: `Delete_ReservedNumber`, `Call_LogEvent_ReservedNumberConsumed`
- Final: `Call_LogEvent_Success`, `Respond_Success`, `Respond_Failure`

---

## Environment variable references

| EV | Expression |
|---|---|
| EV_DocCentralV3_SharePointSite | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| EV_DocCentralV3_lstSviPredmeti | `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')` |
| EV_DocCentralV3_lstAppConfig | `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')` |
| EV_DocCentralV3_lstAuditLog | `@parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')` |
| EV_DocCentralV3_docDokumenti | `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')` |
| EV_DocCentralV3_lstRezervisaniBrojevi | `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')` |

---

## Build order recommendation

Build and test incrementally. Do not build the full flow before testing sub-sections.

Recommended order:
1. Trigger + variables + Log Started → test that LogEvent works.
2. Add validation checks → test each validation failure.
3. Add App Config reads → test document type and year validation.
4. Add number acquisition branches → test both paths against child flows.
5. Add Svi predmeti Create item → test item creation.
6. Add main file create + metadata → test file upload.
7. Add attachment loop → test with 1, then 3 attachments.
8. Add AssignPermissions call → test permission path.
9. Add reserved number delete → test delete path.
10. Add Catch scope with compensation → test compensation (TC-012).
11. Run full TC-001 through TC-005 end-to-end.

---

## Known open items — must resolve before building

| Item | Document |
|---|---|
| All Svi predmeti internal column names | `metadata-field-mapping.md` |
| All Dokumenti library internal column names | `metadata-field-mapping.md` |
| App Config keys: document types, active year, org unit group | `metadata-field-mapping.md` |
| Stanje Choice option exact string `Zavedeno` | `metadata-field-mapping.md` |
| customFields to column mapping | `metadata-field-mapping.md` |
| FAILED_PARTIAL status for Svi predmeti | `create-document-orchestration.md` — compensation replaces this |
| EV_DocCentralV3_docDokumenti exact path format | Test manually before building file upload |
