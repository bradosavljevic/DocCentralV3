# 17 - Large List Loading Pattern

## Problem

SharePoint lists can exceed 2,000 items. Direct Canvas queries may have delegation issues, poor performance, SharePoint threshold issues, N+1 lookup problems and excessive memory usage.

## Lists affected

- Partneri
- Svi predmeti
- Dokumenta
- EmailDocuments
- documents for archiving
- AuditLog if displayed
- Rezervisani brojevi if it grows

## Pattern

1. Power Apps calls a read flow.
2. Flow uses SharePoint REST API or Get items with OData filters.
3. Flow selects only required columns.
4. Flow supports pagination where needed.
5. Flow returns JSON.
6. Power Apps parses JSON and builds collection.

## Standard response

```json
{
  "success": true,
  "code": "OK",
  "message": "Data loaded successfully.",
  "dataJson": "[...]",
  "count": 1234
}
```
