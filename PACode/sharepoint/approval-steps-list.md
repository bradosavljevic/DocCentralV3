# SharePoint list: ApprovalSteps

## Status

This list is NOT yet created in the solution. It must be added as a new SharePoint list.

A new environment variable must also be created in the solution:

| Display name | Logical name (suggested) | Purpose |
|---|---|---|
| EV_DocCentralV3_lstApprovalSteps | gpdoccen_EV_DocCentralV3_lstApprovalSteps | ApprovalSteps list name |

This EV must be confirmed and created before flows referencing this list can be deployed.

## Purpose

Tracks each approval step for each document in the approval process.
This list is the system of record for approval state.

Each row represents one approval step in a document's approval chain.
A document with three sequential approvers will have three rows in this list.

## List name

Suggested display name: `ApprovalSteps`
Suggested internal name: `ApprovalSteps`

## Column schema

Columns listed below with suggested internal names. All internal names are UNKNOWN until
the list is created and confirmed. Use suggested names as the baseline.

| # | Display name | Suggested internal name | Type | Required | Notes |
|---|---|---|---|---|---|
| 1 | Title | Title | Single line of text | Yes | Auto-generated step label |
| 2 | DocumentItemId | DocumentItemId | Number | Yes | ID of item in Svi predmeti |
| 3 | DelovodniBroj | DelovodniBroj | Single line of text | Yes | Registry number of document |
| 4 | StepNumber | StepNumber | Number | Yes | Sequence position, 1-based |
| 5 | AssigneeType | AssigneeType | Choice | Yes | User or Group |
| 6 | AssigneeUserEmail | AssigneeUserEmail | Single line of text | No | Required if AssigneeType = User |
| 7 | AssigneeGroupId | AssigneeGroupId | Single line of text | No | Entra group object ID if AssigneeType = Group |
| 8 | AssigneeGroupName | AssigneeGroupName | Single line of text | No | Display name of group |
| 9 | StepStatus | StepStatus | Choice | Yes | Current status of this step |
| 10 | OutcomeByEmail | OutcomeByEmail | Single line of text | No | Email of user who responded |
| 11 | OutcomeByDisplayName | OutcomeByDisplayName | Single line of text | No | Display name of responder |
| 12 | OutcomeAt | OutcomeAt | Date and Time | No | UTC timestamp of response |
| 13 | Comments | Comments | Multiple lines of text | No | Approver comments on response |
| 14 | InitiatorEmail | InitiatorEmail | Single line of text | Yes | Email of document initiator |
| 15 | CorrelationId | CorrelationId | Single line of text | Yes | Flow run correlation ID |
| 16 | NotificationSent | NotificationSent | Yes/No | Yes | Whether notification was sent for this step |

## Choice field values

### AssigneeType

```
User
Group
```

### StepStatus

```
Pending
Approved
Rejected
Skipped
```

`Skipped` is set on remaining steps when a rejection occurs earlier in the chain.

## Indexes (recommended)

For query performance, create the following indexes on the list:

| Column | Reason |
|---|---|
| DocumentItemId | Frequent filter: get all steps for a document |
| StepStatus | Frequent filter: get pending steps |
| AssigneeUserEmail | Frequent filter: get steps for current user |
| AssigneeGroupId | Frequent filter: get steps for user's groups |

## Permissions

- Service account: Full Control (reads and writes all steps)
- Users: Read — filtered by item-level permissions or by Canvas App filter only

Item-level permissions on ApprovalSteps items are NOT broken by default.
The Canvas App filter (`AssigneeUserEmail = gblCurrentUserEmail` OR group membership)
provides the logical access restriction for the "documents waiting for my approval" view.

If SharePoint-level restriction is needed (so users cannot query other users' approval steps
from SharePoint directly), item-level permission breaks should be applied when each step is
created. This adds complexity — decision deferred. Mark as UNKNOWN.

## Relationship to Svi predmeti

`DocumentItemId` links this list to `Svi predmeti`.
There is no SharePoint lookup column relationship — the join is done in flow logic and
Canvas App filters to avoid lookup column delegation issues.

## Lifecycle

Steps are created by `CF_DocCentralV3_SendForApproval`.
Steps are updated by `CF_DocCentralV3_ProcessApprovalResponse`.
Steps are never physically deleted — they remain as a complete audit trail of the approval history.

If a document is sent for approval multiple times (after rejection and resubmission), each new
approval round creates new step records. Previous rounds retain their `Rejected` / `Skipped` state.
Identifying the "current" round: filter by `StepStatus = Pending`. If none, look for the most
recent round by `OutcomeAt` date. Round number tracking is not explicitly modeled — consider
adding a `RoundNumber` column if needed (currently not in schema — UNKNOWN).

## Open items

| Item | Status |
|---|---|
| EV logical name — to be registered in solution | NOT DONE |
| List to be created in SharePoint | NOT DONE |
| Item-level permission breaks per step | DECISION PENDING |
| RoundNumber column for multi-round approval tracking | UNKNOWN — not in current schema |
| Maximum steps per document | No hard limit — application logic enforced |
