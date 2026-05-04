# File upload pattern: CF_DocCentralV3_CreateDocument

## Overview

All document files are uploaded to the root of the Dokumenti document library.
No folders are created. Every file receives a unique system-generated name.
The original file name is preserved in metadata.

---

## File name structure

### Main document

```
{SafeDelovodniBroj}_{guid()}_{sanitizedOriginalFileName}
```

Example:
- DelovodniBroj: `42/2026`
- SafeDelovodniBroj: `42-2026`
- Original file name: `ugovor o saradnji.pdf`
- Sanitized: `ugovor o saradnji.pdf`
- System file name: `42-2026_3f2504e0-4f89-11d3-9a0c-0305e82c3301_ugovor o saradnji.pdf`

### Attachment (prilog)

```
{SafeDelovodniBroj}_PRILOG_{guid()}_{sanitizedOriginalFileName}
```

Example: `42-2026_PRILOG_7c9e6679-7425-40de-944b-e07fc1f90ae7_prilog1.pdf`

The `_PRILOG_` segment distinguishes attachments from the main document in file name browsing,
without relying on metadata.

---

## SafeDelovodniBroj construction

The delovodniBroj (e.g. `42/2026`) contains a forward slash that is not safe in file names.
Apply these transformations to produce SafeDelovodniBroj:

| Character | Replacement |
|---|---|
| `/` | `-` |
| `\` | `-` |
| `:` | `-` |
| `*` | (remove) |
| `?` | (remove) |
| `"` | (remove) |
| `<` | (remove) |
| `>` | (remove) |
| `|` | (remove) |
| `#` | (remove) |
| `%` | (remove) |
| Two or more consecutive spaces | single space |
| Leading/trailing spaces | (trim) |
| Leading/trailing dots | (trim) |

After transformation, `42/2026` → `42-2026`.

Power Automate expression (nested replace chain):
```
trim(
  replace(
    replace(
      replace(
        replace(
          replace(
            replace(
              replace(
                replace(
                  replace(
                    replace(
                      replace(
                        replace(
                          replace(
                            variables('varDelovodniBroj'),
                            '/', '-'),
                          '\\', '-'),
                        ':', '-'),
                      '*', ''),
                    '?', ''),
                  '"', ''),
                '<', ''),
              '>', ''),
            '|', ''),
          '#', ''),
        '%', ''),
      '  ', ' '),
    '.', '')
)
```

Note: the `replace('  ', ' ')` only collapses one level of double-spaces.
If multiple consecutive spaces are possible, add a second replace step.
The final `trim()` removes leading and trailing spaces.

---

## Original file name sanitization

The original file name (from the Canvas App) is sanitized before appending to the system name.
Apply the same character rules as SafeDelovodniBroj above, but with one addition:

- Do NOT replace spaces with underscores — preserve the original spacing for readability.

**File extension preservation:** Ensure the extension (`.pdf`, `.docx`, etc.) is preserved
through all sanitization steps. The replace chain above does not affect file extensions.

If the sanitized original file name becomes empty after all replacements (e.g. a file named `???`):
- Fall back to: `dokument.bin` as a safe default name.
- Log a Warning with the original file name.

---

## Folder path for SharePoint Create file

All files go to the library root. In the SharePoint Create file action:

| Field | Value |
|---|---|
| Site Address | `@parameters('gpdoccen_EV_DocCentralV3_SharePointSite')` |
| Folder Path | `@parameters('gpdoccen_EV_DocCentralV3_docDokumenti')` |
| File Name | `variables('varMainFileSystemName')` |
| File Content | `base64ToBinary(variables('varMainFile')?['fileContentBase64'])` |

The Folder Path field must contain the library's URL path segment (not the display name).
The environment variable `EV_DocCentralV3_docDokumenti` should contain the URL-relative path
to the library root, e.g. `/sites/DocCentralV3/Dokumenti`.

**Do not prepend `/` if the EV value already starts with `/`.**
Test the exact format required by the Create file action for the specific SharePoint site.

---

## Accessing file item ID after Create file

After the **SharePoint — Create file** action, the file item ID is available at:
```
outputs('Create_MainFile')?['body']?['ItemId']
```

This integer ID is used for:
- Updating file metadata via Update file properties (ID field).
- Passing to CF_DocCentralV3_AssignPermissions (documentLibraryFileIds array).

Store it immediately in a variable (`varMainFileId`) before proceeding.

---

## File content decoding

Canvas App passes file content as Base64 strings.
The SharePoint Create file action requires binary content.
Convert using:
```
base64ToBinary(variables('varMainFile')?['fileContentBase64'])
```

For attachments inside the loop:
```
base64ToBinary(items('Apply_to_each_Attachments')?['fileContentBase64'])
```

**Do not** use `decodeBase64()` — that produces a string, not binary content.
`base64ToBinary()` is the correct function for file uploads in Power Automate.

---

## File size limits

| Limit | Value | Notes |
|---|---|---|
| Power Automate action payload | 100 MB (large file support) | Default: 25 MB. Enable chunked transfer for large files. |
| Base64 encoded content | Approximately 1.33× the raw file size | A 10 MB file ≈ 13.3 MB Base64 string |
| Power Apps canvas payload | ~2 MB default | Canvas App may have lower limits for what it can pass to a flow |
| SharePoint file size | Up to 250 GB per file | Effectively unlimited for this use case |

**Practical recommendation for v1:** Limit files to 10 MB each in the Canvas App validation
before calling this flow. This keeps the Base64 payload within ~13 MB, well within Power Automate limits.

If larger files are needed: implement chunked upload via SharePoint REST API large file upload pattern.
This is not in scope for v1.

---

## Uniqueness guarantee

The `guid()` function in Power Automate generates a UUID v4.
The probability of two GUIDs colliding is effectively zero for any practical volume.
Even if two users upload a file with the same original name for the same document
(same delovodniBroj), their system file names will differ because each has a unique guid().

There is no need to check for existing files before upload.

---

## Naming in the Apply to each loop

Inside the attachments loop (`Apply_to_each_Attachments`), the guid() function is called fresh
for each iteration. Each attachment gets a unique GUID in its system name.

Variables set inside Apply to each do not persist their values between iterations automatically.
Use Compose actions inside the loop and reference their outputs immediately in the same iteration.

If you need to track file IDs from the loop, append each ID to `varCreatedFileIds` using:
```
union(variables('varCreatedFileIds'), array(int(outputs('Create_AttachmentFile')?['body']?['ItemId'])))
```

Apply the Set variable — varCreatedFileIds with this expression after each Create file action
inside the loop.

---

## Error handling for file operations

| Error | Action |
|---|---|
| Create main file fails | Terminate Failed → Catch scope. Fatal. |
| Update main file metadata fails | Log Warning. Non-fatal. Continue. |
| Create attachment file fails | Log Warning. Non-fatal. Skip to next attachment. |
| Update attachment metadata fails | Log Warning. Non-fatal. Continue. |
| Sanitized original name is empty | Fall back to `dokument.bin`. Log Warning. Continue. |

The main file is the only fatal upload. The document is not valid without it.
Attachment failures are degraded-state — the document exists, the attachment is missing.
