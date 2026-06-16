# Document Generation in Flows (Word / Excel / PDF)

Reference for generating branded documents and converting to PDF. The connector facts here are the ones that aren't in the docs or are easy to get wrong.

## Connector matrix — what converts what

| Need | Action | Input | Output | Notes |
|---|---|---|---|---|
| Fill a Word template | Word Online (Business) **Populate a Microsoft Word template** (`CreateFileItem`) | template in a Graph library + `dynamicFileSchema/*` | docx **bytes** | doesn't save the result anywhere |
| Word → PDF | Word Online (Business) **Convert Word Document to PDF** (`GetFilePDF`) | a file **already saved** in a Graph library (OneDrive/SharePoint) + `format` | PDF | does NOT take raw bytes → save first |
| Office → PDF | OneDrive **Convert file** (`ConvertFile`) | a file in the connection's **OneDrive** | PDF | source formats: doc(x), xls(x), ppt(x), rtf, odt… **not HTML** |
| Add Excel rows | Excel Online (Business) **Add a row** (`AddRowV2`) | workbook table (by name or id) | — | works on OneDrive or SharePoint-hosted workbooks |
| SharePoint → PDF | **(none)** | — | — | SharePoint connector has no convert; use Word/OneDrive |
| HTML → PDF | **(none, without premium)** | — | — | OneDrive convert rejects HTML; Encodian/Muhimbi/Plumsail are premium |

**Implication:** to PDF a SharePoint-hosted Word doc, use `GetFilePDF` (Word connector reaches SharePoint). To PDF an Excel sheet, it must pass through a OneDrive `ConvertFile` (so the working file must live in a OneDrive the connection owns).

## Word content controls

- Fields are keyed by the content control's **`w:id`**, passed as **flat** params `dynamicFileSchema/{w:id}` (e.g. `dynamicFileSchema/400008`). NOT keyed by tag/alias, NOT a nested object. (Discover ids by mapping one field in the designer and reading the saved param, or by inspecting `word/document.xml`.)
- Picture content controls take `{"$content-type":"image/png","$content":"<base64>"}`. If a record has no image, pass a **1×1 transparent PNG** placeholder — an empty `$content` makes the connector reject the whole template with a misleading `400 "file selezionato non contiene elementi del modello"`.
- **Repeating content controls are NOT supported** by the connector (officially). A repeating section also produces duplicate `w:id`s, which then make PDF conversion fail (`406 cannotOpenFile`). So you cannot build a dynamic-row table via Populate.

### Variable-length lists in Word (workaround for no-repeating)

Use a 2-column table with **one data row**, where each cell holds a **multi-line plain-text content control** (`<w:text w:multiLine="1"/>`). The flow fills:
- left control = `join(items, '\n')`  (e.g. `@join(body('Select_Voci'), decodeUriComponent('%0A'))`)
- right control = `join('✔' per item, '\n')`

The connector renders each `\n` as a `<w:br/>` **within a single paragraph** (not separate rows/paragraphs).

**Alignment trap:** because both columns are one paragraph of line-broken text, a symbol glyph (✔, ✓) gets a different default line height than the text → the symbol column **drifts downward** and looks like it has extra rows. Fix: set **exact line spacing** identical on both content-control paragraphs:

```xml
<w:pPr><w:spacing w:line="320" w:lineRule="exact" w:before="0" w:after="0"/>...</w:pPr>
```
(320 twips = 16pt; 1pt = 20 twips.) Exact line height forces every line to the same height → 1:1 alignment.

**Consequence:** no per-row zebra striping is possible here (it's one paragraph, not rows). Per-row backgrounds require real table rows = Excel or HTML, which conflicts with keeping a Word letterhead. Don't promise zebra on a Populate-based Word checklist.

## Excel specifics

- `AddRowV2` references the table by **name** (stable across file swaps) or by the GUID id the Excel Online service assigns (NOT the openpyxl `id`/`displayName`; the service rewrites it — often `{00000000-000C-0000-FFFF-FFFF00000000}` for a single-table workbook). Prefer the **name**.
- The Excel Online Workbook API **rejects a header-only (empty) table** built by openpyxl: `FileCorruptTryRepair / unsupportedWorkbook`. It needs ≥1 data row. Options: ship a sentinel row and delete it after populating (`DeleteItem` by key column), or accept one blank row.
- Excel→PDF inherits the worksheet **print settings**. A default sheet prints the used range tiny in the top-left of A4. Set page layout **fit-to-width / fit-to-page**, print area, margins, and (for logos) put images in cells or the print header/footer.

## Templates: host them where you can update them

- Host Word/Excel templates in a **SharePoint document library** you can read/write via Graph (as a licensed user), not in a personal OneDrive you may not own.
- **Overwrite via Graph `PUT .../root:/path:/content` to the same path → the driveItem id is unchanged** → the flow (which references the id) picks up the new template with **zero flow changes**. This makes template iteration safe and reversible.
- Reference templates by **driveItem id** (`01...`), which is stable; if you re-create the file (new id), update the flow's `file` param.

## Email attachments

```jsonc
"emailMessage/Attachments": [
  { "Name": "@{concat('Doc_', <code>, '.docx')}", "ContentBytes": "@body('Populate_Doc')" }
]
```
Pass **raw bytes** `@body('Action')`. **Never** `@{base64(body('Action'))}` — the connector then attaches the base64 STRING as the file (≈+33% size, unopenable; a "corrupt" PDF that starts with `JVBERi0x` is base64 of `%PDF`). Action `inputs` look identical in both cases, so verify by opening the attachment.

## Misc

- CMYK JPEGs must be converted to **RGB** before embedding in HTML/Office or PDF rendering (colors break / fail otherwise).
- To preview any docx as the flow will render it: upload to SharePoint and `GET /drives/{drive}/items/{id}/content?format=pdf` (Graph) — same engine the connectors use.
