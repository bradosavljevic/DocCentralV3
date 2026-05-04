# Power Automate Rules — DocCentral

## 1. Purpose

This document defines rules for Power Automate flows in DocCentral.

Power Automate acts as the backend/API layer for the Canvas app.

---

## 2. Flow responsibilities

Power Automate should handle:

- Create document
- Update document
- Register document
- Reserve/generate `DelovodniBroj`
- Archive document
- Apply SharePoint permissions
- Read/transform data when needed
- Audit and error logging

---

## 3. Standard flow structure

Every important flow should use this structure:

```text
Scope - Try
Scope - Catch
Scope - Finally
Respond to Power App or flow
```

### Try

Business logic.

### Catch

Runs if Try fails.

Should:

- collect error details
- write error log
- prepare standardized error response

### Finally

Optional cleanup/logging.

---

## 4. Standard response

Flows called by Power Apps must return standardized response.

Use:

- `templates/flow-response-template.md`

Required response fields:

- `success`
- `statusCode`
- `message`
- `data`
- `error`
- `correlationId`

---

## 5. Naming rules

Action names must be readable.

Avoid names like:

```text
Compose
Compose_2
Condition_5
Apply_to_each_7
```

Prefer:

```text
Compose - Build response payload
Condition - Document can be registered
Apply to each - Permission recipients
```

---

## 6. Connection references

Connection references must be documented.

For each flow, document:

- connector
- connection reference logical name
- expected account/service account
- environment-specific values

---

## 7. Environment variables

Environment variables must be documented.

For each variable, document:

- display name
- schema name
- purpose
- expected value
- environment-specific behavior

---

## 8. Registry number logic

The flow responsible for registration must implement concurrency-safe logic.

See:

- `architecture/concurrency-design.md`

Mandatory:

- server-side number allocation
- unique constraint protection
- retry handling
- logging failed attempts
- standardized response

---

## 9. Error handling

Every critical flow must log errors.

Recommended fields are defined in:

- `templates/error-log-template.md`

Critical errors include:

- SharePoint update failure
- duplicate `DelovodniBroj`
- permission assignment failure
- App Config update failure
- invalid user access
- validation failure
- flow timeout

---

## 10. Retry policy

Retry must be explicitly considered.

Recommended:

- Use default retry for safe read operations where acceptable.
- Use controlled custom retry for concurrency-sensitive writes.
- Do not blindly retry non-idempotent operations without idempotency key/correlation id.

---

## 11. Power Apps timeout awareness

Power Apps-triggered flows should return quickly.

For long operations:

- create a request item
- return request id
- process asynchronously
- notify or refresh status later

Avoid long synchronous operations that can cause 504 Gateway Timeout.

---

## 12. SharePoint writes

All SharePoint writes should be performed under service account where business rules require standard users to be Read Only.

Writes should include:

- validation before write
- error handling
- logging
- standardized response

---

## 13. Permission operations

Permission operations must be controlled and logged.

For item/folder/file permission updates:

- validate recipients
- remove empty addresses
- isolate invalid users/groups where possible
- log failures
- avoid malformed semicolon recipient strings

---

## 14. Flow inventory documentation

Each flow should eventually have a dedicated markdown file containing:

- flow name
- purpose
- trigger
- input parameters
- actions summary
- connectors
- connection references
- environment variables
- SharePoint lists/libraries touched
- response payload
- error handling
- known issues
