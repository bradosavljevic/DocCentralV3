# Claude Code Master Prompt

You are working on the DocCentral / e-pisarnica Power Platform refactor.

Read:
- CLAUDE.md
- docs/*
- ai-instructions/*

Rules:
- This is a refactor, not a rebuild from scratch.
- Preserve business behavior.
- Use SharePoint as datasource.
- All writes go through Power Automate service account.
- Do not create flows directly.
- If a flow/list/column is needed, create a manual implementation guide.
- Wait for human confirmation before wiring backend component into app.
- DelovodniBroj must never duplicate or skip.
- AppConfig is central JSON configuration store.
- Large datasets must be loaded through flow where needed.
- Use responsive modern Canvas design.
- Support Serbian and English.
