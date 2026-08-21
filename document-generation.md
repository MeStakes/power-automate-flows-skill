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
- ✅ **Content controls inside anchored text boxes are populated**, and so are **picture controls placed as absolutely-positioned `wp:anchor`** objects. Confirmed on a template whose entire layout is floating text boxes over a full-page background — the connector does not require body-level or table-level controls. This is what makes a designer's graphics-only file usable (see next section).
- Picture content controls take an **object**: `{"$content-type":"image/png","$content":"<base64>"}`. A JSON **string** with the same content is rejected ("not a PNG or JPG"). The maker UI flattens these params into strings when it saves, so **assert the shape before patching**:
  ```python
  for k in ("dynamicFileSchema/900001", "dynamicFileSchema/900002"):
      v = populate_action["inputs"]["parameters"][k]
      assert isinstance(v, dict) and "$content" in v, f"{k} flattened to string by a UI save — re-pull/fix first"
  ```
- If a record has no image, pass a **1×1 transparent PNG** placeholder — an empty `$content` makes the connector reject the whole template with a misleading `400 "file selezionato non contiene elementi del modello"`.
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

## Turning a designer's file into a populatable template

A file that came from a graphic designer typically has **zero content controls** — so the Word connector can fill nothing. Don't send it back; inject the controls with a script. What such a file looks like, and what the script must handle:

- **All the graphics are one full-page image in the header** (often an EMF wrapping a bitmap), `behindDoc="1"`, page-sized. The body holds only floating text boxes with placeholder text.
- Each text box is an `mc:AlternateContent`: `mc:Choice` (DrawingML `wps`) **plus** `mc:Fallback` (VML) **with the text duplicated**. Keep only the `mc:Choice` and drop the fallback — the Word Online + Graph pipeline never renders the VML, and two copies of the text means ambiguity about which one got populated.
- Key the injection on **`wp:docPr/@id`** of each anchor (stable inside the file), and assign your own `w:id`s in a readable block (e.g. `500001…500025` text, `900001…` pictures). Then **self-check**: reopen the produced docx, validate the XML and assert the set of `w:id`s is exactly what you intended.
- Give injected picture controls a real 1×1 placeholder image so the template is valid on its own.
- **Boxes that sit on top of drawn shapes need `<a:noFill/>`.** A designer's white-filled box over a printed radio circle hides the circle; with the fill removed the mark lands inside it.
- The value you write matters: **Arial has no U+2714 (✔)** — write `X`. Check the glyph exists in the template's font before promising a checkmark.
- Keep the calibration knobs at the top of the script (font, bold, keep/drop the grey "fill me in" shading, picture position/size in EMU). Designer files always need a second pass.

Iterate by PUTting the rebuilt template over the **same** SharePoint path → same driveItem id → no flow change (see below).

## Resolution and page size (why a PDF looks wrong)

- **Grainy PDF = the template's background bitmap, not the conversion.** A designer's EMF often wraps an uncompressed screen-resolution bitmap (e.g. 596×842 px stretched to A4 = **72 dpi**); a print-quality template is 2480×3508 = **300 dpi**. Diagnose by inspecting the **image XObject's pixel size in the output PDF** — if it reads `596 x 842`, the source is the problem and no converter setting will help. Fix = replace the background with a 300 dpi asset.
- **An SVG background in a docx is rasterized anyway.** Embedding `asvg:svgBlip` (with a PNG in `a:blip r:embed`) is valid and future-proof, but the Office renderer uses the **PNG fallback** — verify against the produced PDF and size the PNG for the dpi you actually need.
- **All templates merged into one PDF must share a page size.** A Letter template (`pgSz 12240x15840`) among A4 ones (`11900x16840`) is ~1000 twips shorter → ~3 fewer 16pt rows → a spurious final page with a single row, plus mixed page formats in the merged file. `python-docx` **defaults to Letter**: set `pgSz` explicitly, and re-check the content column widths (A4's usable width is narrower, so a column set that fit Letter can overflow by a few twips).
- **Measure page counts, don't eyeball them:** populate the template the way the flow does, upload to SharePoint, convert with Graph `?format=pdf` (the same Office engine the connectors use), count with `pypdf`. Also verify the **Y step between rows is uniform** — one wrapped line changes the step and everything below drifts.

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
Pass **raw bytes** `@body('Action')`. **Never** `@{base64(body('Action'))}` — the connector then attaches the base64 STRING as the file (≈+33% size, unopenable; a "corrupt" PDF that starts with `JVBERi0x` is base64 of `%PDF`). Action `inputs` look identical in both cases, so verify by opening the attachment (download the run's `inputsLink` and decode `$content`).

**Inline images in the HTML body:** a small logo embedded as a `data:` URI works and travels with the mail (keep it to a couple of KB — a signature wordmark, not a print asset), and regenerating the base64 from the file at build time keeps it in sync. But **Outlook desktop classic may block `data:` images**, showing the `alt` text instead — if the logo must always render, host it on a public https URL and reference that. Decide with the user; don't assume.

## The connector's docx is unreadable in Word — ship the PDF

The Word `CreateFileItem` ("Populate") output docx has **absolute relationship Targets** (`Target="/word/header1.xml"`, `/word/media/...`, `/customXml/item1.xml`) where the source template has relative ones. Everything else about the file is fine, but **Word desktop refuses it** ("unreadable content"), and Word Online tells the user to open it in the desktop app. It happens with every template, and the connector also inflates the customXml parts.

**Consequence:** never deliver the Populate output as a `.docx`. Convert it and attach the PDF.

**Validate a docx from the CLI (macOS):** `textutil -convert txt file.docx`. It is as strict as Word — the connector output fails with "The file isn't in the correct format", the same file with relative Targets converts fine. Handy for telling "the file is broken" apart from "the flow sent the wrong bytes".

## Misc

- CMYK JPEGs must be converted to **RGB** before embedding in HTML/Office or PDF rendering (colors break / fail otherwise).
- To preview any docx as the flow will render it: upload to SharePoint and `GET /drives/{drive}/items/{id}/content?format=pdf` (Graph) — same engine the connectors use.
