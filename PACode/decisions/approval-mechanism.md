# Decision: Approval Mechanism

## Decision

Approval state is tracked in a dedicated SharePoint list (`ApprovalSteps`).
SharePoint is the system of record for all approval data.

The Power Automate Approvals connector may be used optionally as a notification delivery mechanism (email card, Teams adaptive card), but it is not the source of truth and must not be the only approval tracking layer.

## Rationale

- The Canvas App must show documents waiting for the current user's approval — this requires a queryable SharePoint data source filtered by user or group.
- SharePoint item-level permissions must reflect approval participation — this requires explicit permission assignment on the SharePoint item, not just a PA Approvals task.
- Sequential multi-step approval requires ordered step records with clear state transitions.
- Group approval (first-to-respond wins) requires a record that can be marked as resolved by any group member.
- The `Stanje` field on the main document item must be updated by the flow, not by a PA Approvals callback alone.
- Rejected approval must set `Stanje = Odbijeno` and return the document to the initiator — this is a structured SharePoint write operation.

## ApprovalSteps list design

List name (suggested): `ApprovalSteps`
Environment variable: UNKNOWN — to be added to solution as `EV_DocCentralV3_lstApprovalSteps`

### Columns

| Display name | Internal name (suggested) | Type | Required | Notes |
|---|---|---|---|---|
| Title | Title | Single line of text | Yes | Auto or step label |
| DocumentItemId | DocumentItemId | Number | Yes | ID from Svi predmeti |
| DelovodniBroj | DelovodniBroj | Single line of text | Yes | Registry number |
| StepNumber | StepNumber | Number | Yes | Sequence position (1-based) |
| AssigneeType | AssigneeType | Choice | Yes | User, Group |
| AssigneeUserEmail | AssigneeUserEmail | Single line of text | No | If AssigneeType = User |
| AssigneeGroupId | AssigneeGroupId | Single line of text | No | Entra group ID if AssigneeType = Group |
| AssigneeGroupName | AssigneeGroupName | Single line of text | No | Group display name |
| StepStatus | StepStatus | Choice | Yes | Pending, Approved, Rejected, Skipped |
| OutcomeByEmail | OutcomeByEmail | Single line of text | No | Who responded |
| OutcomeByDisplayName | OutcomeByDisplayName | Single line of text | No | Responder display name |
| OutcomeAt | OutcomeAt | Date and Time | No | When response was recorded |
| Comments | Comments | Multiple lines of text | No | Approver comments |
| InitiatorEmail | InitiatorEmail | Single line of text | Yes | Document initiator |
| CorrelationId | CorrelationId | Single line of text | Yes | Flow run correlation |
| NotificationSent | NotificationSent | Yes/No | No | Whether notification was dispatched |

### StepStatus choice values

```
Pending
Approved
Rejected
Skipped
```

## Approval process rules

### Single-user approval
1. Flow creates one ApprovalStep record with `StepNumber = 1`, `AssigneeType = User`.
2. Flow sends notification to the user (email via Outlook, optionally PA Approvals card).
3. Flow updates document `Stanje` to `U odobravanju`.
4. Flow assigns the user Read permission on the SharePoint document item (break inheritance if needed).
5. User views the document in the Canvas App (documents waiting for approval screen).
6. User triggers approval or rejection via Canvas App → flow `CF_DocCentralV3_ProcessApprovalResponse`.
7. Flow updates the ApprovalStep record with outcome.
8. Flow updates document `Stanje` to `Odobreno` or `Odbijeno`.
9. If `Odbijeno`: flow returns item permission to initiator, notifies initiator.

### Sequential multi-step approval
1. Flow creates all ApprovalStep records up front (StepNumber 1..N), all with `StepStatus = Pending`.
2. Only StepNumber 1 is activated (notification sent, permission granted) at creation time.
3. After step N approval, flow looks for the next step (StepNumber N+1).
4. If found and `StepStatus = Pending`, flow activates it (notification + permission).
5. If no next step exists after an approval, document transitions to `Odobreno`.
6. Any rejection at any step terminates the process and sets document to `Odbijeno`.

### Group approval (first-to-respond wins)
1. Flow creates an ApprovalStep with `AssigneeType = Group`.
2. Flow sends notification to all group members (via Office 365 Groups connector: list members, send individual emails or one group email).
3. First group member to respond triggers `CF_DocCentralV3_ProcessApprovalResponse`.
4. Flow checks that the step is still `Pending` before accepting the response.
5. If already resolved (race condition), flow returns a message to the second responder without changing state.
6. Outcome recorded with the specific user's identity.

## Canvas App integration

- The approval screen queries `ApprovalSteps` filtered by `AssigneeUserEmail = currentUser().email` OR by group membership.
- Group membership check: Canvas App reads the user's Entra groups (via Office365Groups connector) on startup and stores in `colUserGroups`.
- Filter: show items where `StepStatus = Pending` AND (`AssigneeUserEmail = gblCurrentUserEmail` OR `AssigneeGroupId in colUserGroupIds`).
- Document details are loaded by joining on `DocumentItemId` from `Svi predmeti`.

## Flow responsibilities

| Flow | Approval responsibility |
|---|---|
| CF_DocCentralV3_SendForApproval | Create ApprovalStep records, send notifications, update Stanje, assign permissions |
| CF_DocCentralV3_ProcessApprovalResponse | Validate step is Pending, record outcome, update Stanje, activate next step or close process |

## Permission model for approval

- When a document enters approval: item-level inheritance is broken, approver (user or group) is granted Read access.
- When approved/rejected: permission is updated — approver access can be retained for audit or revoked per client configuration (to be read from App Config).
- Initiator always retains access to their own documents.

## Open items

| Item | Status |
|---|---|
| ApprovalSteps list environment variable | UNKNOWN — not yet created in solution |
| App Config key for approval permission retention policy | UNKNOWN |
| App Config key for PA Approvals notification toggle | UNKNOWN |
| Multi-language approval email template | UNKNOWN |
