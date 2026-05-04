# Decision: Registry Number Concurrency Strategy

## Problem

Multiple users may attempt to register a document at the same moment.
The registry number (`DelovodniBroj`) must be unique per registry book per year.
Generating the next number by reading and writing the counter non-atomically creates a race condition where two flows could read the same counter value, generate the same number, and produce a duplicate.

## Decision

Use **optimistic locking via SharePoint ETag / If-Match** on the App Config item that stores the current registry book counter.

On conflict (HTTP 412 Precondition Failed), the flow retries with a fresh read.
The flow never returns a duplicate number.
The flow never blocks indefinitely — retry attempts are bounded.

## How the registry book is stored

The active registry book configuration is an item in the `App Config` list.
The item stores the current counter (next number to assign) and the active year.

Assumed App Config structure for registry book (exact keys UNKNOWN until AppConfig.csv is provided):

| App Config key (UNKNOWN) | Assumed purpose |
|---|---|
| UNKNOWN_ActiveYear | The currently active registry year (e.g. 2026) |
| UNKNOWN_RegistryBookCounter | The next DelovodniBroj sequence number |
| UNKNOWN_RegistryBookFormat | Format string for number display (e.g. `{counter}/{year}`) |
| UNKNOWN_RegistryBookId | SharePoint item ID of the active registry book record |

Until exact keys are confirmed, the flow design uses placeholder names prefixed with `APPCONFIG_`.

## Concurrency-safe flow algorithm

### Step 1 — Read active registry book

- Read the App Config item for the active registry book.
- Capture the item's `@odata.etag` value.
- Capture the current counter value.

### Step 2 — Calculate next number

- `nextNumber = currentCounter + 1`
- Format `DelovodniBroj` according to format string from App Config.
- Example: `42/2026`

### Step 3 — Attempt atomic update (If-Match)

- Send a PATCH request to update the counter on the App Config item.
- Include header: `If-Match: <captured-etag>`
- If SharePoint accepts the update (HTTP 200): the number is reserved. Proceed.
- If SharePoint returns HTTP 412 (Precondition Failed): another flow updated the item first. Go to Step 4.
- If SharePoint returns any other error: go to error handling.

### Step 4 — Retry on conflict

- Increment retry counter.
- If retry counter exceeds `MaxRetries` (suggested: 5): fail with error. Log to AuditLog. Return error response to Canvas App.
- Wait a short random interval (suggested: 500ms–1500ms jitter) to reduce thundering herd.
- Return to Step 1 with a fresh read.

### Step 5 — Uniqueness check

After successfully reserving the number, verify there is no existing item in `Svi predmeti` with the same `DelovodniBroj`.
This is a safety net check, not the primary concurrency guard.
If a duplicate is found (should not happen under correct optimistic locking): log error, fail the operation, do not create the document.

### Step 6 — Return reserved number

- Return the reserved `DelovodniBroj` and the formatted display value to the calling flow (`CF_DocCentralV3_CreateDocument`).
- The counter in App Config has already been incremented atomically at this point.

## Error handling

| Condition | Behavior |
|---|---|
| Conflict on first attempt | Retry (up to MaxRetries) |
| Retry limit exceeded | Log to AuditLog (Severity: Error, EventType: GenerateRegistryNumber, Status: Failed). Return structured error to Canvas App. |
| Non-conflict SharePoint error | Log to AuditLog. Return structured error. |
| Duplicate found in Svi predmeti | Log to AuditLog (Severity: Critical). Return structured error. Do not create document. |

## Audit log events

| Event | EventType | Severity | Status |
|---|---|---|---|
| Starting number generation | GenerateRegistryNumber | Info | Started |
| Conflict detected, retrying | GenerateRegistryNumber | Warning | Retried |
| Number successfully reserved | GenerateRegistryNumber | Info | Success |
| Retry limit exceeded | GenerateRegistryNumber | Error | Failed |
| Duplicate detected in Svi predmeti | GenerateRegistryNumber | Critical | Failed |

## Response to Canvas App

Success:
```json
{
  "success": true,
  "message": "Delovodni broj je uspešno generisan.",
  "delovodniBroj": "42/2026",
  "counterValue": 42,
  "correlationId": ""
}
```

Failure:
```json
{
  "success": false,
  "message": "Nije moguće generisati delovodni broj.",
  "errorCode": "REGISTRY_NUMBER_GENERATION_FAILED",
  "correlationId": ""
}
```

## Config values sourced from App Config

| Parameter | Source | Default (suggested) |
|---|---|---|
| MaxRetries | App Config (UNKNOWN key) | 5 |
| RetryJitterMs | App Config (UNKNOWN key) | 500–1500ms random |
| RegistryNumberFormat | App Config (UNKNOWN key) | `{counter}/{year}` |

## Constraint on Canvas App

Canvas App must never calculate or increment the registry number itself.
The number is always generated server-side by `CF_DocCentralV3_GenerateRegistryNumber` and returned to the Canvas App as part of the `CF_DocCentralV3_CreateDocument` response.
