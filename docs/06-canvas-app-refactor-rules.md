# 06 - Canvas App Refactor Rules

Refactor existing app. Do not rebuild from scratch unless explicitly instructed.

## UI

- Minimal visual redesign.
- Existing users should recognize the app.
- Full responsive layout is required.
- Serbian and English are required.
- Use modern controls and Fluent UI style where possible.
- App may have its own compact menu/header even when embedded in SharePoint.

## SharePoint write

Canvas app must not directly write to SharePoint. Forbidden against SharePoint:
- Patch
- Remove
- UpdateIf
- SubmitForm

Allowed:
- Patch local collections
- UpdateContext
- Set
- ClearCollect from flow response
- read from SharePoint only where delegable/performant

## Flow calls

Every critical `Flow.Run` must be wrapped in `IfError`, check response success, display user-friendly message and log/surface technical details.

## Performance

- Reduce App.OnStart.
- Avoid live search against SharePoint on every keystroke.
- Use DelayOutput.
- Avoid cross-screen control references.
- Avoid nested galleries unless justified.
- Use flow-based read for large lists.

## scrCodes

`scrCodes` may be redesigned because it is large and hard to maintain. Goal: easier dictionary management, single-dictionary Excel import/export, schema validation and Power Automate update flow.
