# Power Automate build notes: CF_DocCentralV3_ProcessApprovalResponse

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_ProcessApprovalResponse |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint, CR_DocCentralV3_Outlook, CR_DocCentralV3_Office365Groups |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAuditLog |

---

## JSON array step update — critical pattern

Power Automate has no native "update object in array" operation.
The update requires:
1. Find the target element with `filter()`.
2. Update it with `setProperty()`.
3. Reassemble the array in order with `concat()`.

Use multiple named Compose actions rather than nesting everything in one expression.
Complex nested expressions are impossible to debug in Power Automate when they fail.

**Step-by-step breakdown:**

```
[Compose: Find_TrenutniKorak]
filter(json(variables('varKoraciJson')), equals(item()?['stepNumber'], variables('varTrenutniKorak')))
→ returns array of matching steps (should be 1)

[Compose: Get_TrenutniKorakObj]
first(outputs('Find_TrenutniKorak'))
→ single step object

[Compose: Build_UpdatedStep]
setProperty(setProperty(setProperty(setProperty(
  outputs('Get_TrenutniKorakObj'),
  'status', triggerBody()?['text_outcome']),
  'resolvedBy', triggerBody()?['text_responderEmail']),
  'resolvedAt', utcNow()),
  'comments', triggerBody()?['text_comments'])

[Compose: Steps_Before]
filter(json(variables('varKoraciJson')), less(item()?['stepNumber'], variables('varTrenutniKorak')))

[Compose: Steps_After]
filter(json(variables('varKoraciJson')), greater(item()?['stepNumber'], variables('varTrenutniKorak')))

[Compose: Full_Updated_Array]
concat(outputs('Steps_Before'), array(outputs('Build_UpdatedStep')), outputs('Steps_After'))

[Set variable: varUpdatedKoraciJson]
string(outputs('Full_Updated_Array'))
```

---

## Marking remaining steps Skipped (rejection)

After building varUpdatedKoraciJson (current step = Rejected), update all steps where stepNumber > varTrenutniKorak.

Option A — use `map()` if available:
```
[Compose: Steps_To_Skip]
filter(json(variables('varUpdatedKoraciJson')), greater(item()?['stepNumber'], variables('varTrenutniKorak')))

[Compose: Skipped_Steps]
map(outputs('Steps_To_Skip'), setProperty(item(), 'status', 'Skipped'))

[Compose: Resolved_Steps]
filter(json(variables('varUpdatedKoraciJson')), lessOrEquals(item()?['stepNumber'], variables('varTrenutniKorak')))

[Compose: Final_Rejection_Array]
concat(outputs('Resolved_Steps'), outputs('Skipped_Steps'))

[Set variable: varUpdatedKoraciJson]
string(outputs('Final_Rejection_Array'))
```

Option B — if `map()` is not available in the environment: use Apply to each with Append to array.
Build a varSkippedArray by looping over the steps after the current step and appending
`setProperty(item, 'status', 'Skipped')` to a new array variable.

Test Option A first in the expression editor before committing to it.

---

## Building IstorijaJson history entry — avoid string interpolation

Do not build the history JSON entry as a raw string with `concat()` — comments may contain
double-quotes that break the JSON.

Instead, build it as a Compose object expression:

```
[Compose: Build_HistoryEntry]
{
  "round": variables('varCurrentRound'),
  "stepNumber": variables('varTrenutniKorak'),
  "outcome": triggerBody()?['text_outcome'],
  "byEmail": triggerBody()?['text_responderEmail'],
  "byName": triggerBody()?['text_responderDisplayName'],
  "at": utcNow(),
  "comments": triggerBody()?['text_comments']
}
```

Then append:
```
[Compose: Updated_Istorija]
concat(json(variables('varIstorijaJson')), array(outputs('Build_HistoryEntry')))

[Set variable: varUpdatedIstorijaJson]
string(outputs('Updated_Istorija'))
```

---

## Three PATCH body variants — use Switch, not nested Conditions

The flow has three distinct PATCH bodies (Rejection, NextStepActivated, ProcessComplete).
Use a **Switch** action on `variables('varNextAction')` to route to the correct PATCH body.

Cases:
- `DocumentRejected` → Branch A body
- `NextStepActivated` → Branch B body
- `ProcessComplete` → Branch C body

Each case contains one Compose for the PATCH body JSON string, then the PATCH action.
The PATCH action itself can be placed after the Switch in a shared location by storing
the body in a variable `varPatchBody` set inside each Switch case.
Then one shared PATCH action reads `variables('varPatchBody')`.

---

## Case-insensitive email comparison for authorization

Power Automate expressions:
```
equals(toLower(triggerBody()?['text_responderEmail']), toLower(variables('varOdobravalacEmail')))
```

For group authorization, use `contains()` on the email array (case handling depends on
whether Office365Groups returns emails in consistent case — test with actual tenant).

If case sensitivity is an issue: normalize all emails with `toLower()` before comparison.

---

## 412 retry pattern — maximum one retry

The approval 412 retry is intentionally limited to ONE attempt.

Unlike the registry number flow (5 retries, jitter delay), approval 412 conflicts almost always
indicate that another user already resolved the step. Re-reading the item and checking
`KoraciOdobravanjaJson` status confirms this.

Pattern:
```
[After PATCH_ApprovalOutcome]
Condition: statusCode = 412
Yes:
  [Get_SviPredmeti_Item_Retry]
  [Compose: Retry_StepStatus]
  first(filter(json(outputs('Get_SviPredmeti_Item_Retry')?['body']?['UNKNOWN_KoraciJson']), equals(item()?['stepNumber'], variables('varTrenutniKorak'))))?['status']
  
  Condition: Retry_StepStatus ≠ 'Pending'
  Yes → Respond STEP_ALREADY_RESOLVED, Terminate Succeeded
  No → Refresh varDocETag from retry read, execute PATCH_Retry (duplicate PATCH action)
  
  Condition after PATCH_Retry: statusCode not in [200, 204]
  Yes → Respond STATUS_UPDATE_FAILED, Terminate Succeeded
```

---

## Environment variable references

| EV | Expression |
|---|---|
| EV_DocCentralV3_SharePointSite | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| EV_DocCentralV3_lstSviPredmeti | `@parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti')` |
| EV_DocCentralV3_lstAuditLog | `@parameters('gpdoccen_EV_DocCentralV3_lstAuditLog')` |

---

## Action naming convention

| Purpose | Name |
|---|---|
| Get document item | `Get_SviPredmeti_Item` |
| Get item (retry after 412) | `Get_SviPredmeti_Item_Retry` |
| Find current step | `Find_TrenutniKorak`, `Get_TrenutniKorakObj` |
| Build updated step | `Build_UpdatedStep` |
| Build arrays | `Steps_Before`, `Steps_After`, `Steps_To_Skip`, `Skipped_Steps` |
| Full updated arrays | `Full_Updated_Array`, `Final_Rejection_Array`, `Updated_Istorija` |
| History entry | `Build_HistoryEntry` |
| PATCH | `PATCH_SviPredmeti_ApprovalOutcome` |
| PATCH (retry) | `PATCH_SviPredmeti_ApprovalOutcome_Retry` |
| List group members (auth) | `List_CurrentGroup_Members` |
| List group members (notify next) | `List_NextStep_GroupMembers` |

---

## Checklist

- [ ] INVALID_OUTCOME validated before any SharePoint reads
- [ ] All 8 Svi predmeti approval fields stored in variables (null-safe)
- [ ] NOT_IN_APPROVAL, APPROVAL_STATE_INVALID, STEP_ALREADY_RESOLVED all validated before auth
- [ ] Authorization: user comparison is toLower(); group check queries Office365Groups
- [ ] Group auth API failure is Warning (non-fatal) with audit flag
- [ ] KoraciJson update uses named Compose chain (no single-expression nesting)
- [ ] Remaining steps marked Skipped on rejection
- [ ] IstorijaJson history entry built as object expression (not string concat)
- [ ] varCurrentRound computed from IstorijaJson before appending
- [ ] PATCH uses IF-MATCH with varDocETag; X-HTTP-Method: MERGE
- [ ] 412: re-read, check step status, retry once max
- [ ] Switch/Condition on varNextAction routes to correct PATCH body
- [ ] NextStepActivated: AssignPermissions and email (both non-fatal)
- [ ] ProcessComplete/DocumentRejected: initiator email (non-fatal)
- [ ] All UNKNOWN internal column names resolved before building
- [ ] Flow inside DocCentralV3 solution
