# Prompt: Refactor Screen

Refactor one screen only.

Before changing:
1. Explain current behavior.
2. Identify issues.
3. Propose changes.
4. Confirm business behavior remains unchanged.

Rules:
- no direct SharePoint writes
- no cross-screen control references
- use responsive layout
- use IfError around Flow.Run
- preserve business logic
