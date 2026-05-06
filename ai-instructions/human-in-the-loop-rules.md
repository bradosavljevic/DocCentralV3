# Human In The Loop Rules

Claude Code must not assume backend components exist.

When a new backend component is required:
1. Claude writes manual implementation guide.
2. Human implements it.
3. Human tests it.
4. Human returns final technical details.
5. Claude updates Canvas app/documentation.

Manual implementation required for Power Automate flows, SharePoint lists, libraries, columns, views, permissions, connection references and environment variables.
