# Skill: Power Automate Backend Patterns

## Cilj

Power Automate je backend servisni sloj za poslovne operacije nad SharePoint-om i dokumentima.

## Obavezna pravila

- Svi poslovni write zahtevi iz Canvas aplikacije idu kroz flow.
- Flow mora validirati payload, prava i poslovna pravila.
- Flow mora vratiti strukturisan response aplikaciji.
- Flow mora imati error handling.
- Flow mora imati logovanje za kritične operacije.
- Flow mora raditi pod servisnim nalogom kada upisuje u SharePoint.

## Standardni response format

Svaki flow koji se poziva iz Power Apps-a treba da vraća JSON:

```json
{
  "success": true,
  "code": "OK",
  "message": "Operacija je uspešno završena.",
  "data": {},
  "correlationId": "..."
}
```

Za grešku:

```json
{
  "success": false,
  "code": "VALIDATION_ERROR",
  "message": "Nedostaje obavezno polje.",
  "details": {},
  "correlationId": "..."
}
```

## Flow kategorije

- Power Apps triggered backend flow
- SharePoint triggered flow
- Scheduled flow
- Child flow
- Admin/maintenance flow
- Import flow, trenutno ignorisati za SAP ako nije traženo

## Zabranjeno

- Vraćanje nestrukturisanih tekstualnih poruka bez status koda.
- Dugi flow bez ranog response-a kada Canvas aplikacija čeka rezultat.
- Upis bez validacije.
- Kritične operacije bez audit loga.
