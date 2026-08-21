# Creating a NEW Cloud Flow from the API

Reference for building a flow that doesn't exist yet — no maker portal, no solution import. The edit/deploy loop for an *existing* flow is in [deploy-and-test.md](deploy-and-test.md).

## Who must be logged in (check this FIRST)

**A new flow validates its connections against the CALLER**, not against the connection owner. Creating or activating a flow whose connection references belong to someone else fails.

- Be authenticated as the **identity that owns the connections** (the flow's usual owner / admin account) for `create` and `activate`.
- PATCHing `clientdata` and toggling state on an **existing** flow works fine as any account with write access.

So: if a shared admin account owns the environment's connections, `az login` as that account before creating. Plan for it — it is often a different account from the one you do daily work with.

## 1. Build the clientdata

Same shape as a pulled definition, and **`schemaVersion` must be at the root**:

```python
cd = {
  "properties": {
    "connectionReferences": {
      "shared_commondataserviceforapps": {
        "connection": {"connectionReferenceLogicalName": "<publisher>_sharedcommondataserviceforapps_12864"},
        "runtimeSource": "embedded",
      }
    },
    "definition": definition,          # {"$schema":..., "triggers": {...}, "actions": {...}}
  },
  "schemaVersion": "1.0.0.0",
}
```

Omit it → `0x80060468 "Required property 'schemaVersion' not found"` on POST.

**Reuse existing connection references.** If the connector is already used by another flow in the environment, put that connection reference's **logical name** here and you are done — no `pac solution import`, no manual bind, no human step. Only a connector that has **no** connection reference in the environment yet forces the import path. (Verified with `shared_office365` and `shared_commondataserviceforapps` added to a brand-new flow purely via `clientdata`.)

**Sanity-check `runAfter` before POSTing** — a dangling dependency is accepted at create time and only shows up as a broken run:

```python
for scope, acts in (("root", definition["actions"]), ("if", inner_actions)):
    for name, a in acts.items():
        for dep in (a.get("runAfter") or {}):
            assert dep in acts, f"[{scope}] {name} runAfter -> {dep} does not exist"
```

## 2. POST the workflow record (draft)

```python
rec = {
  "name": "<Area> | <What it does>",
  "description": "...",        # NOTE: the platform copies this into definition.description
  "category": 5,               # 5 = modern (cloud) flow
  "type": 1,                   # 1 = definition
  "primaryentity": "none",     # even for a Dataverse-triggered flow
  "clientdata": json.dumps(cd, ensure_ascii=False),
  "statecode": 0, "statuscode": 1,   # created as DRAFT
}
POST workflows?$select=workflowid       # Prefer: return=representation → workflowid
```

Persist the returned `workflowid` immediately (a file / your script's state) — everything else keys off it.

## 3. Activate

```python
PATCH workflows(<wfid>)  {"statecode": 1, "statuscode": 2}
```

A Dataverse-triggered flow registers its callback here. If the flow was created as a draft and you later patch the definition, you still need the **Flow API stop+start** (deploy-and-test.md §3).

## 4. Add it to a solution

```python
POST AddSolutionComponent
{
  "ComponentId": "<wfid>",
  "ComponentType": 29,                 # 29 = Workflow / cloud flow
  "SolutionUniqueName": "<solution unique name>",
  "AddRequiredComponents": False,
  "DoNotIncludeSubcomponents": False
}
```

Deliberately leaving a flow **outside** any solution is a valid choice for a throwaway test bench (it won't travel with an export) — but say so explicitly, or someone will assume the export is complete.

## 5. Expected diff: live vs your build

After `create` (or any `patch`), a `pull` will NOT match your local build byte for byte. These differences are the platform, not a human editing behind your back:

| Difference | Why |
|---|---|
| `inputs.authentication` removed from triggers and actions | the platform strips it |
| `definition.description` appears | injected from the workflow record's `description` column |
| `subscriptionRequest/filteringattributes` has **extra** columns | seen in the wild; a superset only widens when the flow fires (harmless), but a future `patch` from your build will remove them |

Anything else in the diff is a real change — investigate before overwriting it.

## 6. Script shape that makes this cheap

The pattern that survived many iterations, as subcommands of one script per flow:

```
pull        # live clientdata -> /tmp/<flow>_live.json   (ALWAYS first)
build       # generate the definition -> /tmp/<flow>.json (pure, no network)
create      # POST workflows (draft)
activate    # statecode 1
addsolution # AddSolutionComponent
deploy      # off -> PATCH clientdata -> on -> Flow API stop+start
verify      # pull again and assert the live definition == intent
test        # seed/invoke, poll the run, print statuses + outputsLink
lastrun     # statuses + outputsLink of the most recent run
rollback    # restore the write-once pre-change backup
cleanup     # delete test records by marker
```

`build` being offline and deterministic is what lets you diff intent against live. `rollback` reading a **write-once** backup (saved by the first `pull`, never overwritten) is what makes a risky migration reversible.

When several flows share a field map or a template id, **import it from one module** instead of copying — one source of truth, already validated on the test bench.
