# Approval runtime JSON pattern

## Purpose

Reference document for the two JSON fields stored on each Svi predmeti item
that track the approval process at runtime:

- **KoraciOdobravanjaJson** — the step chain with current status of each step
- **IstorijaOdobravanjaJson** — the full chronological history of all decisions across all approval rounds

Both fields are Multiple lines of text columns in Svi predmeti.
Internal column names are UNKNOWN — display names are confirmed.

---

## KoraciOdobravanjaJson

### Purpose

Stores all steps defined in ProcesConfig for the current approval round, each with its
live status. Reset to a fresh Pending array each time SendForApproval is called.

### Schema

```json
[
  {
    "stepNumber": 1,
    "assigneeType": "User",
    "assigneeEmail": "approver@org.rs",
    "assigneeGroupId": "",
    "status": "Pending",
    "resolvedBy": null,
    "resolvedAt": null,
    "comments": null
  },
  {
    "stepNumber": 2,
    "assigneeType": "Group",
    "assigneeEmail": "",
    "assigneeGroupId": "00000000-0000-0000-0000-000000000001",
    "status": "Pending",
    "resolvedBy": null,
    "resolvedAt": null,
    "comments": null
  }
]
```

### Field definitions

| Field | Type | Values | Notes |
|---|---|---|---|
| stepNumber | integer | 1-based sequential | Must match ProcesConfig step numbers |
| assigneeType | string | `"User"` or `"Group"` | Determines which field is populated |
| assigneeEmail | string | Email or empty string | Set for User steps; `""` for Group steps |
| assigneeGroupId | string | Entra group GUID or empty | Set for Group steps; `""` for User steps |
| status | string | `"Pending"`, `"Approved"`, `"Rejected"`, `"Skipped"` | |
| resolvedBy | string or null | Responder email or null | Null until resolved |
| resolvedAt | string or null | UTC ISO 8601 datetime or null | Null until resolved |
| comments | string or null | Approver comment or null | Null until resolved |

### Status lifecycle

| Status | When set | By which flow |
|---|---|---|
| `Pending` | On initialization | CF_DocCentralV3_SendForApproval |
| `Approved` | On approval | CF_DocCentralV3_ProcessApprovalResponse |
| `Rejected` | On rejection | CF_DocCentralV3_ProcessApprovalResponse |
| `Skipped` | On rejection — all steps after the rejected step | CF_DocCentralV3_ProcessApprovalResponse |

### State after SendForApproval (2-step example)

```json
[
  {"stepNumber": 1, "assigneeType": "User", "assigneeEmail": "step1@org.rs", "assigneeGroupId": "", "status": "Pending", "resolvedBy": null, "resolvedAt": null, "comments": null},
  {"stepNumber": 2, "assigneeType": "Group", "assigneeEmail": "", "assigneeGroupId": "group-guid", "status": "Pending", "resolvedBy": null, "resolvedAt": null, "comments": null}
]
```

### State after step 1 approved

```json
[
  {"stepNumber": 1, "assigneeType": "User", "assigneeEmail": "step1@org.rs", "assigneeGroupId": "", "status": "Approved", "resolvedBy": "step1@org.rs", "resolvedAt": "2026-05-04T10:30:00Z", "comments": ""},
  {"stepNumber": 2, "assigneeType": "Group", "assigneeEmail": "", "assigneeGroupId": "group-guid", "status": "Pending", "resolvedBy": null, "resolvedAt": null, "comments": null}
]
```

### State after step 1 rejected

```json
[
  {"stepNumber": 1, "assigneeType": "User", "assigneeEmail": "step1@org.rs", "assigneeGroupId": "", "status": "Rejected", "resolvedBy": "step1@org.rs", "resolvedAt": "2026-05-04T10:45:00Z", "comments": "Nedostaje prilog."},
  {"stepNumber": 2, "assigneeType": "Group", "assigneeEmail": "", "assigneeGroupId": "group-guid", "status": "Skipped", "resolvedBy": null, "resolvedAt": null, "comments": null}
]
```

---

## IstorijaOdobravanjaJson

### Purpose

Accumulates a permanent record of every individual approval decision across all rounds.
Never reset — only appended. Grows with each ProcessApprovalResponse call and with each
new SendForApproval call (which adds a round-marker entry on resubmission).

### Schema

```json
[
  {
    "round": 1,
    "stepNumber": 1,
    "outcome": "Approved",
    "byEmail": "step1@org.rs",
    "byName": "Korisnik Jedan",
    "at": "2026-05-04T10:30:00Z",
    "comments": ""
  }
]
```

### Field definitions

| Field | Type | Notes |
|---|---|---|
| round | integer | Approval round number. 1 on first send, increments on resubmission. |
| stepNumber | integer | Which step in the chain this decision applies to. |
| outcome | string | `"Approved"`, `"Rejected"`, or `"RoundReset"` (for resubmission markers). |
| byEmail | string | Email of the responder. |
| byName | string | Display name of the responder. |
| at | string | UTC ISO 8601 datetime of the decision. |
| comments | string | Approver's comment. Empty string if none provided. |

### Round marker entry (resubmission)

When SendForApproval is called after a rejection (resubmission), append a round-marker
before resetting KoraciOdobravanjaJson:

```json
{
  "round": 1,
  "stepNumber": 0,
  "outcome": "RoundReset",
  "byEmail": "<initiatorEmail>",
  "byName": "<initiatorDisplayName>",
  "at": "<utcNow()>",
  "comments": "Novi krug odobravanja pokrenut od strane inicijatora."
}
```

`stepNumber: 0` indicates this is a process-level event, not a step-level decision.

### Full example (rejection round 1, resubmission, approval round 2)

```json
[
  {"round": 1, "stepNumber": 1, "outcome": "Approved", "byEmail": "user1@org.rs", "byName": "User Jedan", "at": "2026-04-01T09:00:00Z", "comments": ""},
  {"round": 1, "stepNumber": 2, "outcome": "Rejected", "byEmail": "user2@org.rs", "byName": "User Dva", "at": "2026-04-01T10:00:00Z", "comments": "Nedostaje prilog."},
  {"round": 1, "stepNumber": 0, "outcome": "RoundReset", "byEmail": "inicijator@org.rs", "byName": "Inicijator", "at": "2026-04-03T08:00:00Z", "comments": "Novi krug odobravanja pokrenut od strane inicijatora."},
  {"round": 2, "stepNumber": 1, "outcome": "Approved", "byEmail": "user1@org.rs", "byName": "User Jedan", "at": "2026-04-03T09:30:00Z", "comments": ""},
  {"round": 2, "stepNumber": 2, "outcome": "Approved", "byEmail": "user3@org.rs", "byName": "User Tri", "at": "2026-04-03T10:15:00Z", "comments": ""}
]
```

---

## Round tracking

The current approval round number must be tracked. Two approaches:

**Option A:** Count distinct `RoundReset` entries in IstorijaOdobravanjaJson + 1.
`currentRound = length(filter(json(varIstorijaJson), outcome == "RoundReset")) + 1`

**Option B:** Store a `varCurrentRound` integer, incremented at each resubmission.
Read from a dedicated column on Svi predmeti (UNKNOWN — would require an additional column).

**Recommendation:** Use Option A. No additional column needed.
SendForApproval computes `currentRound` from IstorijaOdobravanjaJson before appending the RoundReset marker.

---

## JSON manipulation in Power Automate

Power Automate does not have native JSON array mutation (no push, no splice).
The pattern to update a specific element in a JSON array is:

### Pattern: Update one element in an array

Goal: update step where `stepNumber = varTrenutniKorak`.

```
varUpdatedKoraciJson = string(
  union(
    filter(json(varKoraciJson), item()?['stepNumber'] != varTrenutniKorak),
    array(
      setProperty(
        first(filter(json(varKoraciJson), item()?['stepNumber'] == varTrenutniKorak)),
        'status', outcome
      )
    )
  )
)
```

**Power Automate expressions used:**
- `filter(array, condition)` — returns elements matching the condition.
- `first(array)` — returns the first element.
- `setProperty(object, key, value)` — returns a new object with the property set.
- `union(array1, array2)` — concatenates arrays (also works on objects, so use with care).
- `string(value)` — serializes to JSON string.

**Warning:** `union()` on arrays in Power Automate returns the union of unique elements,
not simple concatenation. For arrays that may contain duplicate-looking objects,
use `concat()` instead:

```
concat(
  filter(json(varKoraciJson), item()?['stepNumber'] != varTrenutniKorak),
  array(updatedStepObject)
)
```

Then re-sort by stepNumber if order matters (use `sort()` expression or build the array in order).

### Pattern: Append to history array

Goal: append a new entry to IstorijaOdobravanjaJson.

```
varUpdatedIstorijaJson = string(
  concat(
    json(variables('varIstorijaJson')),
    array(
      json(
        concat(
          '{"round":', string(varCurrentRound),
          ',"stepNumber":', string(varTrenutniKorak),
          ',"outcome":"', triggerBody()?['text_outcome'],
          '","byEmail":"', triggerBody()?['text_responderEmail'],
          '","byName":"', triggerBody()?['text_responderDisplayName'],
          '","at":"', utcNow(),
          '","comments":"', replace(triggerBody()?['text_comments'], '"', '\"'),
          '"}'
        )
      )
    )
  )
)
```

**Warning:** String interpolation inside JSON is fragile if comments contain double quotes.
Use `replace(comments, '"', '\\\"')` to escape quotes in comments.
Or build the history entry as a Compose object and use `string()` on it:

```
[Compose — Build_HistoryEntry]
Expression: {
  "round": variables('varCurrentRound'),
  "stepNumber": variables('varTrenutniKorak'),
  "outcome": triggerBody()?['text_outcome'],
  "byEmail": triggerBody()?['text_responderEmail'],
  "byName": triggerBody()?['text_responderDisplayName'],
  "at": utcNow(),
  "comments": triggerBody()?['text_comments']
}

[Compose — varUpdatedIstorijaJson]
Expression: string(concat(json(variables('varIstorijaJson')), array(outputs('Build_HistoryEntry'))))
```

Building the entry as an object expression (not a string) avoids all escaping issues.

---

## Null/empty handling

| Field state | How to detect | How to initialize |
|---|---|---|
| KoraciOdobravanjaJson is null (column never set) | `empty(varKoraciJson)` | Use `'[]'` |
| KoraciOdobravanjaJson is `"[]"` (empty array string) | `equals(length(json(varKoraciJson)), 0)` | Same |
| IstorijaOdobravanjaJson is null | `empty(varIstorijaJson)` | Use `'[]'` |

Always default to `'[]'` before parsing. Never call `json()` on a null value.

```
varKoraciJsonSafe = if(empty(varKoraciJson), '[]', varKoraciJson)
```

---

## ETag / If-Match for race-condition safety

Both SendForApproval and ProcessApprovalResponse use the SharePoint HTTP PATCH
with `If-Match` header to prevent two simultaneous writes to the same document item.

See `PACode/flows/CF_DocCentralV3_GenerateRegistryNumber/concurrency-etag-pattern.md`
for the full ETag pattern.

**Specific to approval flows:**

| Scenario | 412 response handling |
|---|---|
| SendForApproval: another user sent the doc to approval simultaneously | Re-read item. Check Stanje. If now `U odobravanju` → return ALREADY_IN_APPROVAL. |
| ProcessApprovalResponse: two group members respond simultaneously | Re-read item. Check current step status. If no longer `Pending` → return STEP_ALREADY_RESOLVED. |

Maximum retries for approval PATCH: **1** (unlike registry number which retries 5 times).
For approval, a 412 on the second attempt means genuine contention — return STEP_ALREADY_RESOLVED rather than retrying indefinitely.

---

## Stanje values used in approval workflow

| Value | Set by | Meaning |
|---|---|---|
| `Zavedeno` | CF_DocCentralV3_CreateDocument | Document registered, not yet in approval |
| `U odobravanju` | CF_DocCentralV3_SendForApproval | Approval in progress |
| `Odobreno` | CF_DocCentralV3_ProcessApprovalResponse | All steps approved |
| `Odbijeno` | CF_DocCentralV3_ProcessApprovalResponse | Rejected at any step |

Allowed starting states for SendForApproval: `Zavedeno`, `Odbijeno` (resubmission).
Not allowed: `U odobravanju` (ALREADY_IN_APPROVAL), `Arhivirano` (DOCUMENT_ARCHIVED), `Odobreno`.
