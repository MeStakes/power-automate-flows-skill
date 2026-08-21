---
name: power-automate-flows
description: Use when creating, editing, deploying, testing, or debugging Power Automate cloud flows on Dataverse — especially flows that generate Word/Excel/PDF documents, deploy via clientdata patch instead of solution import, create a brand-new flow from the API, or hit trigger-not-firing, non-invocable-trigger, connection-binding, date/timezone, or identity/OneDrive-access problems.
---

# Power Automate Cloud Flows (Dataverse) — Build, Deploy, Test

## Overview

Operate Power Automate **cloud flows** (Dataverse-triggered) from the CLI/API instead of the maker portal: pull the live definition, edit it as JSON, deploy by patching `clientdata`, and verify by triggering a real run and reading its raw outputs. Covers creating a flow from scratch via the API, document generation (Word/Excel → PDF) and the connector/identity gotchas that waste the most time.

**Core principle:** the deployed flow is the source of truth — always pull it live, never trust local solution files; and verify by reading the actual generated files, not by trusting an action's "Succeeded".

**Always pull the latest cloud version first.** Before reading, editing, patching, or reasoning about any flow, GET its current `clientdata` from the environment — even if a local snapshot, an exported solution, or a previous session's JSON is sitting right there. Someone may have edited the flow in the maker UI in between, and the UI silently rewrites definitions (dynamic schema stripped, image params flattened to strings). Skip the pull **only** when the user explicitly says to work from a specific local file or snapshot.

**Tooling assumed:** `pac` (Power Platform CLI), `az` (Azure CLI, for Graph + Dataverse + Flow API tokens), Python (urllib). All operations here are doable headless.

## When to use

- Editing a cloud flow without the maker UI (the UI re-resolves/strips dynamic schema on save).
- **Creating a new flow** (definition, activation, solution membership) from the API.
- Deploying flow-definition changes repeatedly (a fast inner loop) without full solution import.
- A flow shows **ON but never fires** on create/update.
- You need to run a flow whose trigger is **manual / Button / PowerApps V2** from the CLI.
- Generating branded **Word/Excel documents** and converting to **PDF** in a flow.
- A Word template comes from a designer and has **no content controls** at all.
- A generated PDF looks **grainy**, has **mixed page sizes**, or a date is **off by one day**.
- A connector action fails with a cryptic code (`0x80060467`, `0x80060468`, `0x80090487`, `unsupportedWorkbook`, `TriggerInputSchemaMismatch`, 406, NotFound) — see reference files.
- You can't write a template file because it lives in a **OneDrive you don't own**.

## Golden rules (quick reference)

| Situation | Do this |
|---|---|
| Start ANY work on a flow | GET the live `clientdata` from the env first. Local snapshots / exported solutions / last session's JSON are stale by default. Only skip if the user explicitly asked to use a given file. |
| Edit a flow's logic | Pull live `clientdata`, edit JSON, PATCH it back. Don't edit in UI (strips dynamic schema). |
| After PATCHing clientdata | **stop+start via Flow API** to re-register the trigger (statecode toggle alone does NOT). |
| Before a migration/rework patch | Save the pre-change `clientdata` to a **write-once backup file** and ship a `rollback` command. |
| Create a NEW flow | POST `workflows` with `clientdata` carrying root `"schemaVersion":"1.0.0.0"`, then activate, then `AddSolutionComponent`. Be logged in as the **connections' owner** — a new flow validates connections against the caller. |
| Reuse a connector in a new flow | Point `connectionReferences` at a connection reference **already bound** in the env → no solution import needed. Only a genuinely NEW connection reference forces `pac solution import` + manual bind. |
| Run a Button / PowerApps V2 trigger from CLI | Not invocable as-is. Temporarily swap the trigger to `kind:Http`, POST the callback URL, restore in a `finally`, and assert the restored JSON is **identical**. |
| Verify a run | Read each action's `outputsLink` (SAS, no auth) and inspect the real bytes/files. For what was actually *sent* (To, subject, body, attachment), read `inputsLink` — expressions are resolved there, the definition isn't. |
| Email attachment | `ContentBytes: @body('Action')` — **never** `@{base64(...)}` (double-encodes → corrupt). |
| Deliver a generated Word doc | Convert to **PDF**. The Word Populate output docx has absolute relationship Targets → Word desktop calls it unreadable. |
| Host a template | Put it in a **SharePoint** library you control; overwrite via PUT to the same path → same driveItem id → no flow change. Reference by **driveItem id**, not path. |
| Fill a Word template | Content controls keyed by **`dynamicFileSchema/{w:id}`**. No repeating sections (unsupported). Anchored text boxes **are** populated. |
| Template has no content controls (designer file) | Inject them with a script; don't ask the designer. See document-generation.md. |
| Convert to PDF | Word docs → Word `GetFilePDF`; Excel/Office → OneDrive `ConvertFile`. No SharePoint-native convert. No HTML→PDF. |
| Merging PDFs from several templates | Make every template the **same page size** (A4 = `pgSz 11900x16840`). `python-docx` defaults to **Letter**. |
| A DateOnly **UserLocal** column | Pass the value through **raw**. Formatting it from UTC drops a day. `convertFromUtc(..., 'W. Europe Standard Time')` only when you must format for display. |
| Any write (seed/deploy) | Guardrail: assert the target org id before writing. Never touch prod by accident. |

## The deploy + test loop

This is the high-frequency workflow. Full commands, the trigger-reregistration trap, invoking non-invocable triggers, and expected live-vs-build diff noise: **see [deploy-and-test.md](deploy-and-test.md)**.

1. **Pull** live definition: `GET workflows(<id>)?$select=clientdata`.
2. **Edit** the JSON (actions/triggers/connectionReferences).
3. **Deploy**: PATCH `clientdata` (turn off → patch → on), then **Flow API stop+start** (re-registers the trigger).
4. **Test**: seed a trigger record via Dataverse Web API (with a cleanup marker) → poll Flow API for the new run → inspect action statuses → fetch outputs via `outputsLink` → verify files → cleanup by marker.

Creating a flow that doesn't exist yet (record fields, activation, solution membership, who must be logged in): **see [creating-flows.md](creating-flows.md)**.

## Document generation (Word / Excel / PDF)

The connector matrix, content-control mechanics, turning a designer's graphics-only file into a populatable template, resolution/page-size traps, and the alignment and empty-Excel-table traps: **see [document-generation.md](document-generation.md)**.

Key facts that aren't obvious:
- Word **Populate** fills content controls by `w:id`, **including controls inside anchored text boxes and absolutely-positioned picture anchors**. That's what makes a designer's full-page-graphic template usable.
- **No repeating sections** → for variable-length lists use multi-line plain-text content controls (one per column), and set **exact line spacing** identical across columns or symbol glyphs drift.
- Excel→PDF only via OneDrive `ConvertFile`; the sheet's **print settings** decide the PDF.
- Word→PDF (`GetFilePDF`) needs the doc **saved in a Graph library** first (it doesn't take raw bytes).
- A grainy PDF is almost always the **template's background bitmap dpi**, not the conversion. Check the image XObject's pixel size in the output PDF.

## Auth, environments, identities

Why `pac`/`az` drift, who can access which drive, who may create a flow, and the guardrail pattern: **see [auth-environments.md](auth-environments.md)**.

Key facts:
- `pac` and `az` silently point at the wrong env/tenant — verify before every write.
- **Creating/activating a NEW flow requires being the identity that owns its connections**; PATCH/toggle of an existing flow works as any admin.
- An `az` delegated admin token can read/write **SharePoint** libraries but **cannot** access another user's personal **OneDrive** (403). Service accounts often have **no** OneDrive at all.
- A flow's connections may be owned by a different identity than the one you're logged in as — check `connectionreferences`.

## Common mistakes

- Editing the flow in the maker UI "just to check" → it re-saves and strips the dynamic content-control schema and flattens picture params into strings → runs fail with "doesn't contain template elements". Pull/patch via API instead, and **assert picture params are objects** before patching.
- Trusting local `*.json` solution files → they go stale vs the deployed flow. Always pull live.
- Concluding "it works" from action status `Succeeded` → download and open the real output file; rendering/encoding bugs don't fail the action.
- Verifying an email against the flow **definition** instead of the run's `inputsLink` → you never see the resolved values.
- Toggling statecode and assuming the trigger is live → it isn't until you stop+start via Flow API.
- Reading `subscriptionRequest/message` as a bitmask → it's a plain optionset (`4` = Added or Modified, `3` = Modified only).
- Putting a template in a personal OneDrive → you may not be able to update it. Use SharePoint.
- Shipping the connector's docx to a user → it won't open. Ship the PDF.
