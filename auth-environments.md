# Auth, Environments & Identities

Reference for the access problems that block flow work. These are about WHO you are and WHICH environment you're hitting — get them wrong and writes go to the wrong place or fail with permission errors.

## pac and az drift — verify before every write

`pac` and `az` silently keep a previously-selected profile/tenant. Before any write operation, assert the target:

```bash
pac auth list                 # pick the right index
pac auth select --index <N>
pac org who                   # MUST show the expected environment
az account show --query "{user:user.name, tenant:tenantId}" -o json
az account set -s <tenantOrSubId>   # if wrong
```

**Guardrail in scripts:** before seeding/deploying, call Dataverse `WhoAmI` and assert `OrganizationId == <expected>`. Abort otherwise. This is the single best protection against accidentally writing to production.

## Two different "environment" identifiers

- **Org id / org URL** (`https://<org>.crm<N>.dynamics.com`, a GUID): for Dataverse Web API + `pac`.
- **Flow environment id** (a different GUID): for the Flow API (`/environments/{ENV}/flows/...`).

Don't mix them. The Flow API host is region-specific (crm4 → `emea.api.flow.microsoft.com`).

## Who can access which drive (the OneDrive trap)

- An `az` **delegated admin** Graph token can typically **read/write SharePoint** document libraries (Sites scope) — good: host templates there.
- That same token **cannot access another user's personal OneDrive** → `403 Access denied`, or `/drives/{id}` → "malformed or does not represent a valid drive".
- **Service/shared accounts often have NO OneDrive** at all → `/me/drive` → `404 Item not found`.
- A flow's connectors run as the **bound connection's identity**, which may differ from the account that created the connection reference and from the account you're logged in as. So a flow can successfully write a OneDrive at runtime that YOU cannot write from the CLI.

**Find who owns a flow's connections:**
```
GET connectionreferences?$select=connectionreferencelogicalname,connectionid,_createdby_value
# then resolve _createdby_value via systemusers(<id>)?$select=fullname,internalemailaddress
```

**Practical rule:** don't depend on writing a file into a OneDrive you don't own. Host templates/scratch in SharePoint (writable by a licensed user via Graph) or in the OneDrive of the identity you're actually logged in as.

## Connection references & binding

- The flow's `connectionReferences` map a logical connector (`shared_wordonlinebusiness`, `shared_onedriveforbusiness`, `shared_office365`, `shared_commondataserviceforapps`, `shared_sharepointonline`, …) to a connection.
- Adding a connector the flow didn't use = a NEW connection reference = `pac solution import` + a manual **bind** in the maker UI (Solutions → Connection references). Binding generally requires the connection owner's interactive consent — plan for a human step.
- Re-authorizing a connection as a different identity changes what the flow can reach at runtime (e.g. an account with no OneDrive can't be the OneDrive connection's identity for file ops).

## Quick error → cause map

| Error / code | Likely cause |
|---|---|
| `0x80060467` "connection references need connections" on turn-on | A connection reference isn't bound to a connection. |
| `0x80090487` "... not allowed to upload during Create" | Setting an image/file column in a create POST. PATCH after create. |
| `0x80040203` `ActiveUnpublished` on import | A draft exists; publish the flow first. |
| `403 Access denied` / "malformed drive" on Graph | Trying to reach a drive (OneDrive) your token doesn't own. |
| `404 Item not found` on `/me/drive` | The signed-in account has no provisioned OneDrive. |
| `unsupportedWorkbook` / `FileCorruptTryRepair` | Excel Workbook API can't open the file (e.g. header-only table). |
| `406 cannotOpenFile` on PDF convert | Often a Word doc with duplicate `w:id` (repeating section). |
