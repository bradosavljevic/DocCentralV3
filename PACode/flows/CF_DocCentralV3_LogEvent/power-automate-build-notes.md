# Power Automate build notes: CF_DocCentralV3_LogEvent

## Purpose of this document

Practical notes and gotchas for building this flow in the Power Automate maker portal.
Read this alongside `implementation-steps.md` before starting.

---

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_LogEvent |
| Solution | DocCentralV3 |
| Trigger | Power Apps (V2) |
| Connection reference | CR_DocCentralV3_SharePoint |
| Environment variables used | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstAuditLog |

---

## Power Apps V2 trigger — naming behaviour

When you add inputs to the Power Apps V2 trigger, Power Automate assigns internal names
automatically based on the input name and type:

| Input type in trigger | Internal reference prefix |
|---|---|
| Text | `text_` |
| Number | `decimal_` |
| Boolean | `bool_` |
| Date | `date_` |

Example: an input named `eventType` of type Text becomes `triggerBody()?['text_eventType']`
in expressions.

**Critical:** Always verify actual internal names in the expression editor after adding inputs.
The prefix convention above is typical but may vary between Power Automate environments or versions.
Use the dynamic content picker where possible — it shows the actual reference name.

---

## Environment variable references in actions

To use environment variables in SharePoint actions:

- In the **Site Address** field of any SharePoint action, choose the dropdown → enter custom value:
  ```
  @parameters('EV_DocCentralV3_SharePointSite')
  ```
  This references the solution environment variable by logical name.
  If the logical name is `gpdoccen_EV_DocCentralV3_SharePointSite`, the expression is:
  ```
  @parameters('gpdoccen_EV_DocCentralV3_SharePointSite')
  ```

- In the **List Name** field:
  ```
  @parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')
  ```

The exact logical name must match what is registered in the solution. Check the solution's
environment variable definitions for the exact logical name.

If the environment variable fields do not autocomplete in the UI, switch to expression mode
and type the `@parameters()` reference manually.

---

## Connection reference

Every SharePoint action must use `CR_DocCentralV3_SharePoint`, not a personal SharePoint connection.

When adding a SharePoint action:
1. In the connection dropdown, look for a connection tagged with the connection reference name.
2. If it does not appear automatically, go to the solution → Connection references →
   `CR_DocCentralV3_SharePoint` and ensure it is connected.
3. In the flow action, select "Use a connection reference" and pick `CR_DocCentralV3_SharePoint`.

Using a personal connection instead of the connection reference means the flow will fail
when the personal connection owner leaves the organisation.

---

## Try / Catch scope pattern

Power Automate does not have native Try/Catch keywords. The pattern is:

1. Add a **Scope** action. Name it `Try`. Place all main logic inside it.
2. Add a second **Scope** action after the first. Name it `Catch`.
3. Configure the Catch scope's "Run after" settings:
   - Uncheck `is successful`
   - Check `has failed`
   - Check `has timed out`
   - (Optionally check `has been skipped` if relevant)
4. The Catch scope now only runs if the Try scope fails or times out.

To check whether the Try scope succeeded (to decide which response to return):

After both scopes, add a **Condition** action:
- Left: `result('Try')?[0]?['status']`
- Operator: `is equal to`
- Right: `Succeeded`

Yes branch → Respond with success.
No branch → Respond with failure.

---

## Choice columns in SharePoint Create item

For Choice-type SharePoint columns, the **SharePoint — Create item** connector action
presents the column as a dropdown or text field. Enter the choice string directly.

Do **not** wrap the value in a JSON object like `{ "Value": "Info" }` — that is only needed
when using the raw SharePoint REST API. The standard connector handles serialisation.

If the choice string does not exactly match a defined option (including case), SharePoint
will return a `400 Bad Request` error. The Catch scope will handle this.

---

## Number columns — null vs 0

For Number columns (`DocumentItemId`, `ReservedNumberId`, `PartnerId`):

- If the value is `0` (not applicable), consider leaving the field blank/null in the
  Create item action rather than setting it to `0`.
- To conditionally set a field to null: use a **Compose** before Create item to build
  the value conditionally, then use the Compose output in the field.
- Alternatively, accept `0` as the "not set" sentinel and document this in the list's column description.

Recommended approach for v1: accept `0` as the sentinel. This avoids conditional complexity.
If reporting or filtering on these columns becomes important, migrate to null later.

---

## CreatedAt column

The AuditLog list has a custom `CreatedAt` column (Date and Time type).
This is distinct from SharePoint's built-in `Created` system column.

- Set `CreatedAt` to `utcNow()` in the flow.
- The built-in `Created` column is also set automatically by SharePoint to the creation time.
  Both will be approximately the same timestamp.
- Do not attempt to write to the system `Created` column — it is read-only.

If `CreatedAt` does not appear in the Create item action UI (possible for newly created columns),
try refreshing the connection or using the column's internal name in a custom value field.

---

## EnvironmentName field

The `EnvironmentName` column in AuditLog is optional and is left empty in v1.

If you want to populate it in the future, the Power Automate environment ID is available via:
```
workflow().tags.environmentName
```
or
```
workflow().tags.environmentId
```

This is not required for v1 functionality.

---

## SolutionName field

Set the `SolutionName` column to the static string `DocCentralV3` in the Create item action.
This does not need to be read from an environment variable — it is a constant for this solution.

---

## Respond to a PowerApp or flow action

This flow is called as a **child flow** by other flows. The "Respond to a PowerApp or flow"
action is used to return the result.

Output parameter types must match what callers expect:

| Output name | Type | Notes |
|---|---|---|
| success | Boolean | True or False |
| auditLogItemId | Integer | Item ID or 0 |
| correlationId | Text | GUID string |
| message | Text | Error message or empty |

**Important:** All calling flows must define these output names as inputs in their
"Run a Child Flow" action. The names must match exactly (case-sensitive).

If calling flows do not define the outputs, they will receive empty values even if the
child flow sets them.

---

## Anti-pattern: Do not call this flow from itself

The Catch scope of this flow must **never** call `CF_DocCentralV3_LogEvent`.
Doing so creates an infinite loop: a failed write triggers another write attempt, which
also fails, which triggers another, and so on until the flow times out.

If the AuditLog write fails, the Catch scope only composes the failure response and returns.
It does not attempt any further logging.

---

## Saving and versioning

- Save frequently during build.
- After completing the build, save a checkpoint note in the flow description:
  `"v1.0 — initial implementation. CF_DocCentralV3_LogEvent. DocCentralV3 solution."`
- Do not publish flows from outside the solution. Always manage via the solution context.

---

## Checklist before marking as done

- [ ] Flow named exactly `CF_DocCentralV3_LogEvent`
- [ ] Trigger is Power Apps (V2)
- [ ] All 20 trigger inputs defined with correct types
- [ ] `varCorrelationId` initialized — uses input if non-empty, generates GUID otherwise
- [ ] `varCreatedAt` initialized with `utcNow()`
- [ ] Try scope contains the Create item action
- [ ] Catch scope runs only on Try failure / timeout
- [ ] Catch scope does NOT call this flow
- [ ] Both success and failure responses returned via Respond action
- [ ] Response includes: `success`, `auditLogItemId`, `correlationId`, `message`
- [ ] All SharePoint actions use `CR_DocCentralV3_SharePoint` connection reference
- [ ] Site Address uses EV expression (not hardcoded URL)
- [ ] List Name uses EV expression (not hardcoded list name)
- [ ] Flow is inside `DocCentralV3` solution
- [ ] Test cases TC-001 through TC-004 pass (minimum)
