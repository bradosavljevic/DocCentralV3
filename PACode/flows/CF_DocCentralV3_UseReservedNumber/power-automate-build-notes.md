# Power Automate build notes: CF_DocCentralV3_UseReservedNumber

## Purpose of this document

Practical notes and gotchas for building this flow in the Power Automate maker portal.
Read this alongside `implementation-steps.md` and `consumption-pattern.md` before starting.

---

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_UseReservedNumber |
| Solution | DocCentralV3 |
| Trigger | Power Apps (V2) — child flow pattern |
| Called by | CF_DocCentralV3_CreateDocument |
| Connection reference | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstRezervisaniBrojevi, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAuditLog |

---

## Power Apps V2 trigger — input types and prefixes

This flow has 4 trigger inputs:

| Input name | Type in trigger | Internal reference prefix | Full expression |
|---|---|---|---|
| reservedNumberId | Number | `decimal_` | `triggerBody()?['decimal_reservedNumberId']` |
| requestedYear | Number | `decimal_` | `triggerBody()?['decimal_requestedYear']` |
| initiatorEmail | Text | `text_` | `triggerBody()?['text_initiatorEmail']` |
| correlationId | Text | `text_` | `triggerBody()?['text_correlationId']` |

Number inputs come back as decimals — always wrap numeric trigger values in `int()` when using them
in integer contexts:
```
int(triggerBody()?['decimal_reservedNumberId'])
```

Verify actual internal names in the expression editor after adding each input.

---

## Get item vs Get items — which to use for the reserved number lookup

Use **SharePoint — Get item** (single item by ID) for the reserved number lookup, not Get items.
The caller always provides the exact SharePoint item ID.

Get item returns a 404 error (not an empty array) if the item does not exist.
The 404 must be caught using a Condition on the action's status code.

To catch a 404, configure the action's "Configure run after" to include both `is successful`
and `has failed`, then check `outputs('Get_item')?['statusCode']` in the following Condition:
- 404 → handle as RESERVED_NUMBER_NOT_FOUND
- 200 → proceed

Alternatively, wrap the Get item inside a Try scope and handle failures in the Catch scope.
Recommended approach: the Try/Catch scope pattern, which the main Try scope already provides.
If the Get item throws, the Catch scope handles it as a generic RESERVED_NUMBER_VALIDATION_FAILED error.

For a targeted RESERVED_NUMBER_NOT_FOUND response (not generic catch), use "Configure run after" on the Condition that follows Get item.

---

## Terminate vs Respond inside the Try scope

When a validation fails (RESERVED_NUMBER_NOT_FOUND, RESERVED_NUMBER_YEAR_MISMATCH, etc.),
the flow must exit gracefully without triggering the Catch scope.

Pattern:
1. Add **Respond to a PowerApp or flow** with the failure response.
2. Add **Terminate** — Status: **Succeeded** (not Failed).

Using Terminate Succeeded after Respond exits the Try scope cleanly.
The post-scope Condition on `result('Try')?[0]?['status']` will read `Succeeded`,
but at that point the Respond has already been sent. No second Respond is needed.

**Warning:** Do not use Terminate Failed for validation failures. That would trigger the Catch scope,
which would then send a second Respond action — Power Automate does not support two Respond calls
and the flow will error.

---

## Environment variable references

| EV display name | Expression |
|---|---|
| EV_DocCentralV3_SharePointSite | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| EV_DocCentralV3_lstRezervisaniBrojevi | `@parameters('gpdoccen_EV_DocCentralV3_lstRezervisaniBrojevi')` |
| EV_DocCentralV3_lstAppConfig | `@parameters('gpdoccen_EV_DocCentralV3_lstAppConfig')` |
| EV_DocCentralV3_lstSviPredmeti | `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')` |
| EV_DocCentralV3_lstAuditLog | `@parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')` |

If the logical name prefix differs from `gpdoccen_`, verify the exact prefix in the solution's
Environment Variables section.

---

## Multiple Get items actions in one flow

This flow calls Get items (or Get item) on three different lists:
- RezervisaniBrojevi (Get item by ID)
- App Config (Get items with filter)
- Svi predmeti (Get items with filter for duplicate check)

Rename each action clearly to avoid expression ambiguity:
- `Get_RezervisaniBroj` — Get item on RezervisaniBrojevi
- `Get_AppConfig_ActiveYear` — Get items on App Config
- `Get_SviPredmeti_DuplicateCheck` — Get items on Svi predmeti

When referencing outputs in expressions, use the renamed action name:
```
body('Get_AppConfig_ActiveYear')?['value']
body('Get_SviPredmeti_DuplicateCheck')?['value']
outputs('Get_RezervisaniBroj')?['body']
```

---

## Condition for duplicate check — length expression

The Svi predmeti duplicate check uses:
```
length(body('Get_SviPredmeti_DuplicateCheck')?['value'])
```

Compare this with `0` using the `greater than` operator.
If length > 0, a duplicate exists.

Avoid using `empty()` on the value array as it may behave unexpectedly with null vs empty array.
Use `length()` for reliability.

---

## Filter query syntax for Svi predmeti

The OData filter for the duplicate check uses single quotes around the string value:
```
UNKNOWN_InternalColumnName eq '@{variables('varDelovodniBroj')}'
```

The `@{...}` is a Power Automate expression interpolation inside a string field.
In the Filter Query field of Get items, this is written directly — not wrapped in `@{}` at
the top level, only the variable reference inside the string.

If the internal column name for DelovodniBroj in Svi predmeti contains spaces or special characters,
use the internal name (without spaces), not the display name.

---

## Try / Catch scope — run after settings

- Try scope: default.
- Catch scope → Configure run after:
  - Uncheck `is successful`
  - Check `has failed`
  - Check `has timed out`

Post-scope Condition:
```
result('Try')?[0]?['status']  is equal to  Succeeded
```

The Catch scope contains its own Respond action for unhandled errors.
The post-scope Condition Yes branch logs the final Success audit entry.
The post-scope Condition No branch does nothing (Catch already responded).

---

## CF_DocCentralV3_LogEvent calls in this flow

| When | eventType | severity | status | notes |
|---|---|---|---|---|
| Before Try | UseReservedNumber | Info | Started | Always called |
| On RESERVED_NUMBER_NOT_FOUND | UseReservedNumber | Error | Failed | Inside Try, before Terminate Succeeded |
| On RESERVED_NUMBER_YEAR_MISMATCH | UseReservedNumber | Warning | Failed | Inside Try, before Terminate Succeeded |
| On YEAR_NOT_ACTIVE | UseReservedNumber | Warning | Failed | Inside Try, before Terminate Succeeded |
| On RESERVED_NUMBER_ALREADY_USED | UseReservedNumber | Critical | Failed | Inside Try, before Terminate Succeeded |
| After post-scope Condition (Yes) | UseReservedNumber | Info | Success | Validation success log |
| In Catch scope | PowerAutomateError | Error | Failed | Unhandled exception catch |

**Critical:** The Catch scope must NEVER call CF_DocCentralV3_UseReservedNumber.
Only CF_DocCentralV3_LogEvent is called from Catch. Calling this flow from its Catch would create an infinite loop.

---

## Deletion not in this flow — reminder

Do not add a Delete item action for RezervisaniBrojevi in this flow.
Deletion belongs entirely to CF_DocCentralV3_CreateDocument.

If you see a Delete item action being added here during build, stop and re-read `consumption-pattern.md`.

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_UseReservedNumber`
- [ ] Trigger is Power Apps (V2) with 4 inputs: reservedNumberId (Number), requestedYear (Number), initiatorEmail (Text), correlationId (Text)
- [ ] All 5 variables initialized before Try scope
- [ ] varCorrelationId auto-generates GUID when input is empty
- [ ] Log Started called before Try scope
- [ ] Get item on RezervisaniBrojevi uses CR_DocCentralV3_SharePoint and EV expression
- [ ] 404 / item not found handled: RESERVED_NUMBER_NOT_FOUND + Terminate Succeeded
- [ ] varDelovodniBroj and varReservedYear populated from actual internal column names (not UNKNOWN)
- [ ] Year check: varReservedYear vs requestedYear → RESERVED_NUMBER_YEAR_MISMATCH on mismatch
- [ ] App Config active year read and compared with requestedYear → YEAR_NOT_ACTIVE on mismatch
- [ ] Svi predmeti duplicate check → RESERVED_NUMBER_ALREADY_USED with Critical log on duplicate
- [ ] All validation failure branches use Terminate Succeeded (not Terminate Failed)
- [ ] Success Respond does NOT include a Delete item action
- [ ] Catch scope runs on failed/timed out only
- [ ] Catch scope calls LogEvent (PowerAutomateError/Error/Failed)
- [ ] Catch scope does NOT call CF_DocCentralV3_UseReservedNumber
- [ ] Both Respond actions include all 7 output parameters
- [ ] All SharePoint actions use CR_DocCentralV3_SharePoint
- [ ] All EV references use @parameters() syntax
- [ ] All UNKNOWN column names replaced with actual confirmed values before building
- [ ] Flow is inside DocCentralV3 solution
