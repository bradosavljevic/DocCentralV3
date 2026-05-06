# Canvas Coding Standards

This project follows the uploaded reference standard: **2024 Power Apps Coding Standards For Canvas Apps** by Matthew Devaney.

## Naming

Control prefixes: btn, gal, frm, txt, cmb, lbl, con, cmp, ico, img, tgl, dte.

Variables:
- global: `gbl`
- local/context: `loc`
- collections: `col`
- SharePoint collections: `colSp`
- configuration collections: `colCfg`

## Error handling

- Enable Formula-Level Error Management.
- Use `IfError` for Flow.Run and write operations.
- Check flow response before displaying success.
- Do not navigate away after failed submit/update.

## Performance

- Use delegable formulas.
- Cache repeated data in collections.
- Limit collections to required columns.
- Reduce App.OnStart.
- Use `Concurrent` where safe for independent reads.
- Use DelayOutput for text inputs.
- Avoid live cloud search on every keystroke.
- Avoid cross-screen control references.
- Avoid nested galleries unless justified.

## Readability

- Use `With()` for complex formulas.
- Flatten nested Ifs.
- Use consistent logical operators.
- Use `Self` where appropriate.
- Keep comments useful and updated.
