# 19 - Development Environment

## Power Platform environment

Environment ID:

```text
32b4305d-1d34-ec3a-8ed7-f5abd9985d45
```

Tenant ID:

```text
b5f62db0-39d2-4763-9c01-9f250adb3b41
```

## Canvas app

App ID:

```text
d17711d4-1417-4b8d-ac63-df49131264d0
```

Solution ID:

```text
a2ca7e7f-8547-f111-bec6-0022489d3551
```

## Power Apps Studio / edit link

Use this link for Canvas app editing, coauthoring and MCP Canvas work:

```text
https://make.powerapps.com/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/canvas/?action=edit&app-id=%2Fproviders%2FMicrosoft.PowerApps%2Fapps%2Fd17711d4-1417-4b8d-ac63-df49131264d0&solution-id=a2ca7e7f-8547-f111-bec6-0022489d3551
```

## Runtime / play link

Use this link only for running and testing the app as a user:

```text
https://apps.powerapps.com/play/e/32b4305d-1d34-ec3a-8ed7-f5abd9985d45/a/d17711d4-1417-4b8d-ac63-df49131264d0?tenantId=b5f62db0-39d2-4763-9c01-9f250adb3b41&hint=978a7273-8cff-45bb-b033-c4040a8a39dd&sourcetime=1778059741251
```

## Claude Code / MCP Canvas usage rule

Claude Code should use the Power Apps Studio / edit link for development context.

The play link is for runtime testing only.

Claude Code must not assume it can create Power Automate flows, SharePoint lists, SharePoint libraries, SharePoint columns, environment variables or connection references directly through MCP Canvas.

If any backend component is needed, Claude Code must create a manual implementation guide first. The human developer will implement it manually and return the final technical details.
