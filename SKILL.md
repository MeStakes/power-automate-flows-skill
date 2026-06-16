---
name: power-automate-flows
description: Use when creating, editing, deploying, testing, or debugging Power Automate cloud flows on Dataverse — especially flows that generate Word/Excel/PDF documents, deploy via clientdata patch instead of solution import, or hit trigger-not-firing, connection-binding, or identity/OneDrive-access problems.
---

# Power Automate Cloud Flows (Dataverse) — Build, Deploy, Test

## Overview

Operate Power Automate **cloud flows** (Dataverse-triggered) from the CLI/API instead of the maker portal: pull the live definition, edit it as JSON, deploy by patching `clientdata`, and verify by triggering a real run and reading its raw outputs. Covers document generation (Word/Excel → PDF) and the connector/identity gotchas that waste the most time.

**Core principle:** the deployed flow is the source of truth — always pull it live, never trust local solution files; and verify by reading the actual generated files, not by trusting an action's "Succeeded".

**Tooling assumed:** `pac` (Power Platform CLI), `az` (Azure CLI, for Graph + Dataverse + Flow API tokens), Python (urllib). All operations here are doable headless.

## When to use

- Editing a cloud flow without the maker UI (the UI re-resolves/strips dynamic schema on save).
- Deploying flow-definition changes repeatedly (a fast inner loop) without full solution import.
- A flow shows **ON but never fires** on create/update.
- Generating branded **Word/Excel documents** and converting to **PDF** in a flow.
- A connector action fails with a cryptic code (`0x80060467`, `0x80090487`, `unsupportedWorkbook`, 406, NotFound) — see reference files.
- You can't write a template file because it lives in a **OneDrive you don't own**.

## Golden rules (quick reference)

| Situation | Do this |
|---|---|
| Edit a flow's logic | Pull live `clientdata`, edit JSON, PATCH it back. Don't edit in UI (strips dynamic schema). |
| After PATCHing clientdata | **stop+start via Flow API** to re-register the trigger (statecode toggle alone does NOT). |
| Add a NEW component (connection ref, etc.) | Needs `pac solution import` (not just patch) + manual connection bind. |
| Verify a run | Read each action's `outputsLink` (SAS, no auth) and inspect the real bytes/files. |
| Email attachment | `ContentBytes: @body('Action')` — **never** `@{base64(...)}` (double-encodes → corrupt). |
| Host a template | Put it in a **SharePoint** library you control; overwrite via PUT to the same path → same driveItem id → no flow change. Reference by **driveItem id**, not path. |
| Fill a Word template | Content controls keyed by **`dynamicFileSchema/{w:id}`**. No repeating sections (unsupported). |
| Convert to PDF | Word docs → Word `GetFilePDF`; Excel/Office → OneDrive `ConvertFile`. No SharePoint-native convert. No HTML→PDF. |
| Any write (seed/deploy) | Guardrail: assert the target org id before writing. Never touch prod by accident. |

## The deploy + test loop

This is the high-frequency workflow. Full commands and the trigger-reregistration trap: **see [deploy-and-test.md](deploy-and-test.md)**.

1. **Pull** live definition: `GET workflows(<id>)?$select=clientdata`.
2. **Edit** the JSON (actions/triggers/connectionReferences).
3. **Deploy**: PATCH `clientdata` (turn off → patch → on), then **Flow API stop+start** (re-registers the trigger).
4. **Test**: seed a trigger record via Dataverse Web API (with a cleanup marker) → poll Flow API for the new run → inspect action statuses → fetch outputs via `outputsLink` → verify files → cleanup by marker.

## Document generation (Word / Excel / PDF)

The connector matrix, content-control mechanics, the alignment trap, and the empty-Excel-table trap: **see [document-generation.md](document-generation.md)**.

Key facts that aren't obvious:
- Word **Populate** fills content controls by `w:id`; **no repeating sections** → for variable-length lists use multi-line plain-text content controls (one per column), and set **exact line spacing** identical across columns or symbol glyphs drift.
- Excel→PDF only via OneDrive `ConvertFile`; the sheet's **print settings** decide the PDF (set fit-to-page or get a tiny table top-left).
- Word→PDF (`GetFilePDF`) needs the doc **saved in a Graph library** first (it doesn't take raw bytes).

## Auth, environments, identities

Why `pac`/`az` drift, who can access which drive, and the guardrail pattern: **see [auth-environments.md](auth-environments.md)**.

Key facts:
- `pac` and `az` silently point at the wrong env/tenant — verify before every write.
- An `az` delegated admin token can read/write **SharePoint** libraries but **cannot** access another user's personal **OneDrive** (403). Service accounts often have **no** OneDrive at all.
- A flow's connections may be owned by a different identity than the one you're logged in as — check `connectionreferences`.

## Common mistakes

- Editing the flow in the maker UI "just to check" → it re-saves and strips the dynamic content-control schema → runs fail with "doesn't contain template elements". Pull/patch via API instead.
- Trusting local `*.json` solution files → they go stale vs the deployed flow. Always pull live.
- Concluding "it works" from action status `Succeeded` → download and open the real output file; rendering/encoding bugs don't fail the action.
- Toggling statecode and assuming the trigger is live → it isn't until you stop+start via Flow API.
- Putting a template in a personal OneDrive → you may not be able to update it. Use SharePoint.
