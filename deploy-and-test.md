# Deploy & Test Cloud Flows from the CLI/API

Reference for the high-frequency loop: pull → edit → deploy → trigger → verify, without the maker portal.

## Tokens

All API calls use bearer tokens from `az`:

```bash
# Dataverse (workflows table, seed records)
az account get-access-token --resource https://<org>.crm<N>.dynamics.com --query accessToken -o tsv
# Flow API (runs, stop/start)        — host is region-specific: emea/us/...
az account get-access-token --resource https://service.flow.microsoft.com/ --query accessToken -o tsv
# Microsoft Graph (SharePoint files, format=pdf conversion)
az account get-access-token --resource https://graph.microsoft.com --query accessToken -o tsv
```

Region host for Flow API: e.g. crm4 = `https://emea.api.flow.microsoft.com`. The environment is identified by the **Flow environment id** (a GUID, distinct from the org id).

## 1. Pull the live definition (source of truth)

Local solution `.json` files drift. Always read the deployed `clientdata` — **every time, as the first step, even mid-session and even if a snapshot from ten minutes ago is on disk**. The only exception is an explicit user instruction to work from a named file. A maker-UI save between two of your runs rewrites the definition (dynamic schema stripped, image params flattened to strings), so patching a stale copy silently reverts someone else's fix.

```python
# GET workflows(<WF>)?$select=clientdata,statecode,modifiedon  (Dataverse)
cd = json.loads(resp["clientdata"])
actions = cd["properties"]["definition"]["actions"]
triggers = cd["properties"]["definition"]["triggers"]
connrefs = cd["properties"]["connectionReferences"]
```

`clientdata` is a JSON STRING containing `{properties:{definition, connectionReferences}, schemaVersion}`. Edit `definition.actions` / `definition.triggers`, then write the whole thing back.

## 2. Deploy via clientdata PATCH (no re-import)

Patching `clientdata` preserves connection bindings and the dynamic content-control schema that a full `pac solution import` strips (flakily). Cloud flows must be turned off before the definition changes:

```python
# 1) turn off
dv("PATCH", f"workflows({WF})", {"statecode": 0, "statuscode": 1})
# 2) patch definition
dv("PATCH", f"workflows({WF})", {"clientdata": json.dumps(cd, ensure_ascii=False)})
# 3) turn on
dv("PATCH", f"workflows({WF})", {"statecode": 1, "statuscode": 2})
```

**When you MUST use `pac solution import` instead:** adding a NEW component the flow didn't have — most commonly a new **connection reference** (e.g. adding the SharePoint connector). Import side effects to expect:
- Connection bindings are reset → re-bind in UI (Solutions → Connection references).
- The dynamic `dynamicFileSchema` of Word "Populate" actions can be stripped → verify field count after import; if 0, re-deploy the definition via clientdata PATCH.
- `ActiveUnpublished` (0x80040203) blocks import if a draft exists → publish the flow first.

## 3. ⚠️ Re-register the trigger (the #1 time-waster)

After a clientdata PATCH + statecode toggle the flow shows **ON but the Dataverse trigger does NOT fire** — the webhook/callback registration isn't recreated by the statecode toggle alone.

**Fix — stop+start via the Flow API:**

```
POST {flowhost}/providers/Microsoft.ProcessSimple/environments/{ENV}/flows/{WF}/stop?api-version=2016-11-01
POST {flowhost}/providers/Microsoft.ProcessSimple/environments/{ENV}/flows/{WF}/start?api-version=2016-11-01
```

Both return 200; the flow's `state` becomes `Started` and the trigger fires again. **Always run this at the end of a deploy.** Symptom when you forget: you seed a record, wait, and the latest run is still the old one (no new run).

## 4. Inspect a run + read RAW outputs

The authoritative way to verify — no UI, no mailbox access needed:

```
GET {flowhost}/.../flows/{WF}/runs?api-version=2016-11-01&$top=N           # list runs
GET {flowhost}/.../flows/{WF}/runs/{run}/actions?api-version=2016-11-01     # action statuses
```

Each action has `properties.outputsLink.uri` — a **SAS URL (no Authorization header)**. GET it to read the action's output JSON; for file-producing actions (Word Populate, OneDrive Convert) the `body` is the file (often `{"$content-type":..., "$content": "<base64>"}`). Decode and inspect the real bytes.

Status meaning: `Skipped` on a branch action = the `If`/condition went the other way (NORMAL), not an error. A `Delay` inside an `Until` loop is `Skipped` on the iteration that exits — also normal. Only `Failed`/`Faulted`/`TimedOut` are failures; follow their `outputsLink` for the raw connector error body.

**`inputsLink` is what the run actually SENT.** The definition contains `@{...}` expressions; the run's inputs contain the resolved values. To verify an email really went to the right address with a valid attachment, download `properties.inputsLink.uri` of the send action and check `To`, `Subject`, body HTML and the attachment's `$content` (decode it: a PDF must start with `%PDF`, not `JVBERi0x`). Never claim "the mail is right" from the definition.

**Action counts don't match the designer.** `/runs/{id}/actions` enumerates actions nested inside `If_*` / `Scope_*` / `Until_*` too, so it reports more than the top-level count you see when editing. Compare like with like before concluding something was added or lost.

## 4b. Triggers you cannot invoke from the CLI (Button / PowerApps V2)

A **manual** trigger of `kind: Button` or a **PowerApps V2** trigger cannot be started through the Flow API: every body shape returns `TriggerInputSchemaMismatch`.

**Workaround — temporary trigger swap, always restored:**

```python
live = pull()                                        # live clientdata (source of truth)
orig = json.dumps(live["properties"]["definition"]["triggers"], ensure_ascii=False)
try:
    live["properties"]["definition"]["triggers"] = {"manual": {"type": "Request",
        "kind": "Http", "inputs": {"schema": {"type": "object",
        "properties": {"text": {"type": "string"}}}}}}
    patch_clientdata(live)                           # off -> patch -> on (+ stop/start)
    cb = flow("POST", f"flows/{WF}/triggers/manual/listCallbackUrl?api-version=2016-11-01")
    url = cb.get("response", {}).get("value") or cb.get("value")     # <- see below
    post_json(url, {"text": record_id})
finally:
    live["properties"]["definition"]["triggers"] = json.loads(orig)
    patch_clientdata(live)
    back = pull()
    assert json.dumps(back["properties"]["definition"]["triggers"], ensure_ascii=False) == orig, \
        "trigger NOT restored — check it by hand"
```

Two details that cost time:
- `listCallbackUrl` answers `{"response": {"value": "<url>", ...}, "httpStatusCode": 200}` — the URL is in **`response.value`**, not `value`.
- The restore must be in a `finally` **and verified**: leaving a production flow on an HTTP trigger means anyone with the URL can run it. Print the comparison result; don't assume.

Prefer this over "just click Run in the portal" only when you need it scripted/repeatable — a portal run is free and leaves the definition untouched.

## 5. Testing harness pattern (seed → run → verify → cleanup)

```
1. guardrail: WhoAmI → assert OrganizationId == <expected sandbox>   (never write prod)
2. snapshot current run names
3. seed: POST a trigger record via Dataverse Web API, with a marker field (e.g. code="AUTOTEST-...")
4. poll runs until a NEW run (startTime > seed time) reaches a terminal status
5. inspect actions; for failures print the raw outputsLink body
6. download key outputs (docx/pdf) and VERIFY (e.g. convert docx→pdf via Graph and view)
7. cleanup: delete records by marker
```

Keep the harness idempotent and marker-scoped so cleanup is safe.

## 6. Image/File columns can't be set on Create

Setting a Dataverse **image or file column** in the create POST fails: `0x80090487 "... not allowed to upload during Create operation."` Set them with a **PATCH after create** (image columns accept base64 inline on update; file columns need the `$value` upload).

**Consequence for testing a Create-only trigger:** the create fires the flow BEFORE the image exists → the run sees no image. To test image-dependent output, temporarily widen the trigger to also fire on Update, then revert.

## 7. Dataverse trigger `message` values

`subscriptionRequest/message` on the Dataverse trigger (optionset `callbackregistration.message`):

| Value | Meaning |
|---|---|
| 1 | Added (Create) |
| 2 | Deleted |
| 3 | Modified (Update) |
| 4 | Added or Modified |
| 5 | Added or Deleted |
| 6 | Modified or Deleted |
| 7 | Added or Modified or Deleted |

Changing it is a definition edit → deploy + re-register (section 3). Prefer a superset (e.g. 4) for temporary test widening so create still works even if you forget to revert.

## 8. Dataverse trigger semantics you will trip over

- **`message` is not a bitmask.** It's the `callbackregistration.message` optionset (table above). `3` is *Modified only* — a flow set to `3` never fires on create. Use `4` for create+update.
- **`filteringattributes` reacts to a column being PRESENT in the payload**, not to its value changing. PATCHing a record with the *same* value still fires the flow. That's the cheap **backfill** trick: re-PATCH every relevant record with its current values to make a recalculation flow run over the existing data.
- A wider `filteringattributes` set is harmless if the flow recomputes from scratch; it just fires more often.
- **There is no pre-image.** The trigger body has the new values only. So if a lookup is moved or cleared, the flow cannot fix up the *previous* parent record — it will keep stale data. Say this out loud as a known limit instead of pretending the flow is symmetric; the alternatives are a Delete/Change trigger or a scheduled reconcile.
- Scope matters: `Organization` vs `User` decides whether the flow sees records owned by others.

## 9. Dates: DateOnly UserLocal is a trap

Dataverse date behaviours differ per column and the flow must match them:

| Column behaviour | What to do |
|---|---|
| **DateOnly / UserLocal** (stored as local midnight in UTC, e.g. `2026-07-27T22:00:00Z` = 28 July in Rome) | **Pass it through raw** to another UserLocal column. Formatting it (`formatDateTime(...,'yyyy-MM-dd')`) reads the UTC instant and **loses a day** for records created east of UTC. |
| **TimeZoneIndependent** | Already the intended calendar date; format directly. |
| Any date you must **render** into a document (dd/MM/yyyy, split into boxes) | `convertFromUtc(<value>, 'W. Europe Standard Time')` first, then format. |

Symptom of getting it wrong: a document or a synced field showing the day *before* the one the user typed, only for some records. Check the column's behaviour before "fixing" the expression.
