# Power Automate build notes: CF_DocCentralV3_SendForApproval

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_SendForApproval |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint, CR_DocCentralV3_Outlook, CR_DocCentralV3_Office365Groups |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog |

---

## Building KoraciOdobravanjaJson — the Pending step array

Use an **Append to array variable** loop (Apply to each) over varApprovalSteps.
Initialize an empty array variable `varKoraciArray = []` before the loop.

Inside each iteration:
```
[Append to array variable — varKoraciArray]
Value expression:
{
  "stepNumber": items('Build_KoraciOdobravanjaJson_Loop')?['stepNumber'],
  "assigneeType": items('Build_KoraciOdobravanjaJson_Loop')?['assigneeType'],
  "assigneeEmail": items('Build_KoraciOdobravanjaJson_Loop')?['assigneeEmail'],
  "assigneeGroupId": items('Build_KoraciOdobravanjaJson_Loop')?['assigneeGroupId'],
  "status": "Pending",
  "resolvedBy": null,
  "resolvedAt": null,
  "comments": null
}
```

After the loop: `varKoraciJson = string(variables('varKoraciArray'))`

---

## Extracting email list from Office365Groups — List group members

The Office365Groups — List group members action returns a value array.
To extract only email addresses:

Add **Select** action after List group members:
- From: `body('List_Step1_GroupMembers')?['value']`
- Map key: (leave blank — outputs array of values)
- Map value: `item()?['mail']`

The Select output is an array of email strings.
Set varRecipients = output of Select.

If some members have no email (guest accounts), filter: `filter(body('Select_Emails'), not(empty(item())))`.

---

## PATCH URI construction for EV-embedded list name

Switch Uri field to Expression mode:
```
concat(
  '_api/web/lists/getbytitle(''',
  parameters('gpdoccen_EV_DocCentralV3_lstSviPredmeti'),
  ''')/items(',
  string(variables('varDocumentItemId')),
  ')'
)
```

---

## 412 retry pattern

After the PATCH action, add a Condition:
```
equals(outputs('PATCH_SviPredmeti_ApprovalState')?['statusCode'], 412)
```

Yes branch (412):
1. Get_SviPredmeti_Item again (fresh read).
2. Condition: `equals(outputs('Get_SviPredmeti_Item_Retry')?['body']?['UNKNOWN_Stanje'], 'U odobravanju')`
   - Yes: Respond ALREADY_IN_APPROVAL, Terminate Succeeded.
   - No: Refresh varDocETag from retry read, re-execute PATCH (duplicate the PATCH action inside this branch).
3. If second PATCH also fails: Log STATUS_UPDATE_FAILED, Respond failure, Terminate Succeeded.

Keep the retry limited to once. Do not loop indefinitely.

---

## Sending email to multiple recipients — loop vs To field

The Outlook Send an email (V2) action's To field can accept a semicolon-delimited string
for multiple recipients. Alternatively, use Apply to each if per-recipient error isolation is needed.

For non-fatal email delivery: use Apply to each so that one failure does not block others.
Inside the loop, configure "Run after" on a Condition after the email action to catch failures.

---

## Resubmission detection

At Step 5d, check both conditions:
1. `equals(variables('varStanje'), 'Odbijeno')` — document was rejected
2. `not(empty(variables('varExistingIstorija')))` AND `not(equals(variables('varExistingIstorija'), '[]'))`

Only when both are true is this a resubmission. First-time submission always starts with `varIstorijaJson = '[]'`.

---

## Stanje values — confirm before building

Confirm exact strings in the Stanje Choice column before wiring up checks and PATCH body:
- `U odobravanju` — must match exactly (including space, case)
- `Zavedeno`, `Odbijeno`, `Odobreno`, `Arhivirano`

If the Choice column uses different casing or different strings, update all comparisons.

---

## Checklist

- [ ] ProcesConfig reading resolves correctly for test document type
- [ ] PATCH body uses actual internal column names (not UNKNOWN)
- [ ] IF-MATCH header uses varDocETag (not hardcoded `*`)
- [ ] 412 retry limited to 1 attempt
- [ ] Group member email extraction uses Select action
- [ ] Email loop is non-fatal per recipient
- [ ] AssignPermissions call is non-fatal
- [ ] Resubmission appends to IstorijaJson (not replaces)
- [ ] First submission sets IstorijaJson = `'[]'`
- [ ] Flow inside DocCentralV3 solution
