# Power Automate build notes: CF_DocCentralV3_GenerateArchiveBookPdf

## Flow identity

| Property | Value |
|---|---|
| Flow name | CF_DocCentralV3_GenerateArchiveBookPdf |
| Trigger | Power Apps (V2) — called from Canvas App |
| Connection references | CR_DocCentralV3_SharePoint |
| Environment variables | EV_DocCentralV3_SharePointSite, EV_DocCentralV3_lstSviPredmeti, EV_DocCentralV3_lstAppConfig, EV_DocCentralV3_lstAuditLog, EV_DocCentralV3_docExports |

---

## year input — Number type and decimal_ prefix

`year` is Number type. Power Apps V2 trigger uses `decimal_` prefix:
```
int(triggerBody()?['decimal_year'])
```

Store as `varYear` (Integer). Used in:
- OData filter expression (year filter)
- HTML file name
- HTML content (year in header)
- Log messages

---

## HTML string building in Power Automate

Power Automate does not have a template engine. The HTML is built entirely with `concat()` expressions.

**Key constraint:** Power Automate has a 100,000-character limit on string variables and expression outputs. For large archive books with many rows, the accumulated `varHtmlRows` string may approach this limit.

**Limit estimate:** Each HTML row is approximately 300–500 characters. At 5000 documents, varHtmlRows = ~2.5 MB. This exceeds the limit.

**Mitigation strategies:**
1. **For < 2000 documents** (typical case): string concatenation in Apply to each is fine.
2. **For > 2000 documents**: use the Select action to generate rows as an array, then `join()` to combine — this avoids the per-iteration variable update and may be more efficient.
3. **For > 5000 documents**: pagination is required. Generate HTML in chunks or consider splitting the archive book by month.

For v1, document the limit. Recommend the Select + join() approach as the default.

---

## HTML rows — Select + join() approach (recommended)

Instead of Apply to each with string accumulation, use the Select action:

```
[Select: Select_HtmlRows]
From: variables('varAllDocuments')
Mode: Expression
Value:
concat(
  '<tr>',
  '<td>', string(addProperty({}, 'n', 0)?['n']), '</td>',  ← row number workaround — see note
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_DelovodniBroj']),'',string(item()?['UNKNOWN_DelovodniBroj'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '<td>', if(empty(item()?['UNKNOWN_FilingDate']),'',formatDateTime(item()?['UNKNOWN_FilingDate'],'dd.MM.yyyy')), '</td>',
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_Subject']),'',string(item()?['UNKNOWN_Subject'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_PartnerNazivSnapshot']),'',string(item()?['UNKNOWN_PartnerNazivSnapshot'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_DocumentType']),'',string(item()?['UNKNOWN_DocumentType'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_ArhivskiZnak']),'',string(item()?['UNKNOWN_ArhivskiZnak'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '<td>', if(empty(item()?['UNKNOWN_Arhivirano']),'',formatDateTime(item()?['UNKNOWN_Arhivirano'],'dd.MM.yyyy')), '</td>',
  '<td>', replace(replace(replace(replace(if(empty(item()?['UNKNOWN_Arhivirao']),'',string(item()?['UNKNOWN_Arhivirao'])),'&','&amp;'),'<','&lt;'),'>','&gt;'),'"','&quot;'), '</td>',
  '</tr>'
)

[Compose: Build_HtmlRows]
join(body('Select_HtmlRows'), decodeUriComponent('%0A'))
```

**Row number in Select:** The Select action does not expose the current index natively.
Two options:
- Option A: Omit the row number column from the HTML — leave the `<td>` empty, fill it client-side (acceptable for a print document).
- Option B: Use Apply to each with `varRowNumber` — simpler but slower.
- Option C: After joining all rows, the row number is cosmetic only. If the archive book just needs sequential numbers, they can be filled by the CSS using `counter()`. Add `<style>tbody { counter-reset: rownum; } tbody td:first-child::before { counter-increment: rownum; content: counter(rownum); }</style>` and leave the first `<td>` empty.

**Recommended:** Use CSS counter for row numbering. This avoids the Apply to each loop entirely.

---

## UTF-8 BOM in HTML

```
uriComponentToString('%EF%BB%BF')
```

This produces the three-byte UTF-8 BOM sequence.
Prepend as the very first character before `<!DOCTYPE html>`.

**Why needed:** Windows browsers and Excel treat files without BOM as Windows-1252 by default.
The HTML `<meta charset="UTF-8">` tag is read AFTER the parser starts — the BOM tells the parser before it reads anything.

---

## Pagination — @odata.nextLink

The standard SharePoint Get items connector returns up to 5000 items per call.
If `@odata.nextLink` is present in the response, more pages exist.

**Using the standard connector for page 1, HTTP Request for subsequent pages:**

The standard connector response includes:
```
body('Get_ArchivedDocs_Page1')?['@odata.nextLink']
```

For the HTTP Request pages, response format depends on OData version:
- `odata=nometadata`: `body(...)?['value']` for items, `body(...)?['@odata.nextLink']` for next link
- `odata=verbose`: `body(...)?['d']?['results']` for items, `body(...)?['d']?['__next']` for next link

Use `odata=nometadata` (the default for the standard connector) for consistency:
```
Accept: application/json; odata=nometadata
```

**Do Until condition:**
```
or(
  empty(variables('varNextLink')),
  variables('varPaginationDone')
)
```

Set `varPaginationDone = true` inside the loop when `varNextLink` is empty after a page fetch.

**Important:** Do Until maximum iteration count defaults to 60. Set it to the expected maximum number of pages (e.g. 20 for 100,000 documents at 5000/page).

---

## Accumulating documents across pages — union() pattern

```
[Compose: Append_NextPage]
union(variables('varAllDocuments'), outputs('Get_NextPage_Value'))
```

`union()` in Power Automate concatenates two arrays and removes duplicate objects.
For pagination, there should be no duplicates — but union() is the correct function to use
(the standard "append array" approach requires Apply to each which is slow for large arrays).

After all pages: `varTotalCount = length(variables('varAllDocuments'))`

---

## CSS counter for row numbering (alternative to varRowNumber)

Add to the `<style>` section:
```css
tbody { counter-reset: rownum; }
tbody tr td:first-child::before {
  counter-increment: rownum;
  content: counter(rownum);
}
```

Then the first `<td>` in each row is left empty:
```html
<td></td>
```

The browser/print engine fills in the sequential number automatically.
This approach works correctly for printing and PDF export.

---

## Action naming convention

| Purpose | Name |
|---|---|
| Get org name from App Config | `Get_OrgName` |
| Get archived docs page 1 | `Get_ArchivedDocs_Page1` |
| Get subsequent pages | `Get_ArchivedDocs_NextPage` |
| Append page to varAllDocuments | `Append_NextPage` |
| Select HTML rows | `Select_HtmlRows` |
| Join rows | `Build_HtmlRows` |
| Per-field HTML escape | `Escape_DelovodniBroj`, `Escape_Subject`, etc. |
| Date format | `Format_FilingDate`, `Format_ArhiviranoDate` |
| Full HTML document | `Build_HtmlDocument` |
| Create file | `Create_HtmlFile` |

---

## Checklist

- [ ] year parsed as int(decimal_year)
- [ ] App Config org name read; fallback to placeholder if not found
- [ ] Get_ArchivedDocs_Page1: filter by Stanje AND year; $select limits columns
- [ ] Pagination Do-Until: @odata.nextLink checked, max 20 iterations
- [ ] varAllDocuments accumulated with union() across pages
- [ ] varTotalCount = length(varAllDocuments)
- [ ] Empty result: empty-state row rendered; success returned
- [ ] Select + join() used for HTML rows (not Apply to each string concat)
- [ ] CSS counter used for row numbering (or Apply to each varRowNumber — document which)
- [ ] HTML escaping applied to all 6 text fields
- [ ] Null-safe date formatting for FilingDate and Arhivirano
- [ ] UTF-8 BOM prepended as first character
- [ ] Full HTML assembled from archive-book-output.md template
- [ ] Create file: Exports library root, .html extension
- [ ] Log Success with varTotalCount in message
- [ ] Catch scope: log Error, respond failure
- [ ] Flow inside DocCentralV3 solution
