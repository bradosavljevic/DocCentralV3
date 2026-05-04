# Template: Power Automate response

## Success

```json
{
  "success": true,
  "message": "Operacija je uspešno izvršena.",
  "correlationId": "",
  "data": {}
}
```

## Error

```json
{
  "success": false,
  "message": "Operacija nije uspešno izvršena.",
  "errorCode": "",
  "errorMessage": "",
  "correlationId": ""
}
```

## Pravilo

Svaki flow koji Canvas aplikacija poziva mora vratiti standardizovan response.
