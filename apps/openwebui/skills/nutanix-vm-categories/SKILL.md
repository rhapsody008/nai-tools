---
name: nutanix-vm-categories
description: Add, update (swap value), or remove Prism Central categories on Nutanix AHV VMs through the Nutanix v4 MCP server (rhapsody008/ntnx-api-mcp-server fork, bucketed params + AUTO_ETAG), using the fewest tool calls and never sending a wrong ETag. Use whenever the user wants to tag, label, categorize, untag or re-categorize VMs in Prism Central ("put all zy- VMs into ManagedBy:ZY", "remove Env:Dev from these VMs", "change AppTier from Web to App"), or to create/delete the category key:value first. Trigger even for casual phrasing like "tag my VMs" or "label these in Prism".
---

# Nutanix VM category management

This skill works against the forked Nutanix v4 MCP server (`rhapsody008/ntnx-api-mcp-server`). In that server, every `<namespace>_execute` tool takes **declared parameter buckets**, and the server can fetch ETags itself (`AUTO_ETAG`).

The fork was built after a real run in which one ETag was reused across 16 VMs and 10 of the writes failed. The rules below make that mistake impossible and keep the call count close to the minimum.

## Calling convention (all clients)

```
<ns>_execute(
  operation    = "<exact operation name>",
  path_params  = {"extId": "<uuid>"},
  query_params = {"$filter": "...", "$select": "...", "$limit": 100},
  headers      = {"If-Match": "<etag>"},          # usually omit; see ETags below
  request_body = {...}
)
```

- **Put parameters in the buckets, not at the top level.** Strict clients such as Open WebUI drop undeclared top-level keys silently. Keep the `$` prefix on OData names.
- **`unknown_parameter` means a misnamed key.** The server rejects unknown keys with this error, and the error lists the accepted names. Fix the key and retry once.
- **Find the tools in your tool list.** They are deferred in claude.ai, so load them with `tool_search` (query `nutanix vmm categories`). Use the names exactly as listed, since clients add their own prefixes.
- **You don't need `getOperationSchema` for the operations in this skill.** Their shapes are below.

## Reading results correctly

A call is not a success just because the dispatcher returns `ok: true`. Prism errors can come back inside the payload. Check all of these:

1. `ok: false` with `error.code` set. The codes you'll see:
   - `unknown_parameter`: a key is misnamed. Fix it and retry.
   - `read_only_mode`: the server has `READ_ONLY_MODE=true`. Stop and tell the user. Writes need it set to `false`.
   - `auto_etag_failed`: the server couldn't fetch the ETag. Fall back to an explicit ETag (Mode B).
   - `execution_error`: report the detail.
2. `ok: true` but `payload.data.error` is present, or `$dataItemDiscriminator` is `vmm.v4.error.ErrorResponse`. This is a synchronous Prism rejection. For example, `MISSING_ETAG_HEADER` (PLAT-10004) means the If-Match header was absent.
3. `ok: true` with a `TaskReference`. The request was accepted, but the ETag check runs **asynchronously** inside the task. Only the verify step proves the change happened.

## ETags: what is true and what to do

- An ETag belongs to one VM and one version. It looks like `YXBwbGljYXRpb24vanNvbg==:<base64("<version>,1")>`.
  - Versions differ between VMs.
  - Every successful write bumps the version.
  - NKP/CAPX-managed VMs get bumped by their controllers too.
- **Never reuse one VM's ETag on another VM, and never predict the next version.** In the original run, two VMs happened to share version 8. That made reuse look like it worked until the async tasks failed with `VM_ETAG_MISMATCH`.
- **Mode A (preferred): the server handles ETags.** With `AUTO_ETAG=true`, omit `headers` entirely. The server GETs the parent VM and injects `If-Match` right before the write. The model never does per-VM GETs, and the 4–5k-token VM payloads stay out of your context.
- **Mode B (fallback): explicit ETags.** Use this when the server doesn't auto-fetch. GET each target VM with `ahv_getVmById` and send *that VM's* `_etag` in `headers.If-Match`. `$select` is ignored on GET by ID, so each GET returns the full VM.
- **The Batch API (`prism` `submitBatch`) never auto-fetches.** Every batch item needs its own `If-Match` header, so batching only pays off in Mode B.

## Workflow: add or remove a category

### Step 1: Resolve the category (1 call)

```
prism_execute(operation="listCategories", query_params={"$filter": "key eq 'ManagedBy'"})
```

Take the `extId` of the entry whose `value` matches.

If it's missing and the user wants to add it, create it with `createCategory`. Check its body with `getOperationSchema` once first; the expected fields are key, value and description.

### Step 2: Find the targets and pre-filter (1 call)

```
vmm_execute(operation="ahv_listVms",
            query_params={"$filter": "startswith(name, 'zy-')",
                          "$select": "extId,name,categories", "$limit": 100})
```

- **Filtering.** `name`, `powerState`, `cluster/extId` and `host/extId` are filterable. `categories` is not, so read it through `$select`.
- **Paging.** If `totalAvailableResults` is more than 100, page with `$page`.
- **Sanity check.** Before any write, confirm every returned name actually matches the filter. If unrelated VMs appear, the filter didn't reach Prism (a client dropping parameters). **Stop. Do not write.** Tell the user the client is dropping query parameters.
- **Drop VMs that need no change.** For an add, drop VMs that already list the category `extId`. For a remove, drop VMs that don't have it.
- If the target set is non-trivial, state it in one sentence (for example "16 VMs, 7 are NKP-managed"). Ask first only if it looks broader than the user meant.

### Step 3: Write

**Probe with the first target VM, sending no `headers`:**

```
vmm_execute(operation="ahv_associateCategories",        # or ahv_disassociateCategories
            path_params={"extId": "<vm>"},
            request_body={"categories": [{"extId": "<categoryExtId>"}]})
```

- **The result is a TaskReference.** Mode A works. Repeat the same call for each remaining VM, one call per VM, no GETs and no `headers`.
- **The result is `MISSING_ETAG_HEADER` or HTTP 428.** The server isn't auto-fetching. Either `AUTO_ETAG` is off, or the server predates the undeclared-If-Match fix (see "Server requirements" below). Switch to Mode B for every VM, including this one:
  - **3 or more VMs:** GET each target, then send one `submitBatch` call in which each item carries its own ETag:

    ```
    prism_execute(operation="submitBatch", request_body={
      "metadata": {"action": "ACTION", "name": "associate <key>:<value>",
                   "uri": "/api/vmm/v4.2/ahv/config/vms/{extId}/$actions/associate-categories",
                   "shouldStopOnError": false, "chunkSize": 20},
      "payload": [{"data": {"categories": [{"extId": "<cat>"}]},
                   "metadata": {"path": [{"name": "extId", "value": "<vm>"}],
                                "headers": [{"name": "If-Match", "value": "<THIS vm's _etag>"}]}}]})
    ```

    For a remove, use `.../$actions/disassociate-categories`. Check the batch with `getBatchById`, using the `BatchId` from the task's `completionDetails`.
  - **1–2 VMs:** for each VM, GET it and then call `ahv_associateCategories` with `headers={"If-Match": "<that VM's _etag>"}`.
- **The result is `auto_etag_failed`.** Use Mode B for that VM.

### Step 4: Verify (1 call)

Re-run the Step 2 list. Every target should now have the category (for an add) or lack it (for a remove).

**For each miss:** call `prism_execute(operation="getTaskById", path_params={"extId": "<task>"})`.

- **`VM_ETAG_MISMATCH` (VMM-30303):** something changed the VM between the fetch and the write.
  - Mode A: just repeat the write once; the server fetches a fresh ETag.
  - Mode B: retry with `headers.If-Match` set to `YXBwbGljYXRpb24vanNvbg==:` + `argumentsMap.current_etag` from the error.
- **Task still RUNNING:** wait, then verify again. Don't retry yet.

## Workflow: update a value (e.g. Env:Dev → Env:Prod)

AHV has no "replace" action. An update is a disassociate followed by an associate.

1. **Look up both values** with a single `listCategories` call filtered on the key.
2. **List and pick targets:** the VMs that hold the old value.
3. **Disassociate the old value** from every target (Step 3).
4. **Verify once.** This confirms the disassociations finished. Don't skip it: an associate that runs while a disassociate task is still in flight can read a stale ETag and fail.
5. **Associate the new value** with every target (Step 3).
6. **Verify again.**

## Call budget (N = VMs that need a change)

| Path | Calls |
|---|---|
| Mode A, add or remove | tool_search 1 + listCategories 1 + list 1 + N writes + verify 1 = **N + 4** |
| Mode B, 3 or more VMs (batch) | 4 + N GETs + 1 batch + 1–2 polls = **about N + 6** |
| Mode A, update value | **2N + 5** |

For reference, the original 16-VM run took about 50 calls and loaded 16 full VM payloads into context. Mode A takes 20 calls and loads none.

## Server requirements (rhapsody008 fork)

- **`READ_ONLY_MODE=false`.** Required for any write. The deploy default is `true`.
- **`AUTO_ETAG=true`.** Required for Mode A.
- **`STRICT_PARAMS=true`.** Recommended, so mistakes come back as `unknown_parameter` errors instead of silent drops.
- **The undeclared-If-Match fix.** The vmm v4.3 spec doesn't declare `If-Match` on `ahv_associateCategories` and `ahv_disassociateCategories`, but Prism still enforces it. Without the fix, AUTO_ETAG skips those operations and the probe returns `MISSING_ETAG_HEADER`. The skill then falls back to Mode B automatically, so nothing breaks; it's just slower.

## Afterwards, tell the user

- **What changed:** the count of VMs changed, grouped by role or name pattern, plus any repairs and why they were needed.
- **Existing tags are safe.** Associate *adds* categories and disassociate removes only the ones named, so other tags (such as the CAPX cluster category on NKP nodes) remain.
- **NKP/CAPX nodes:** nodes that are replaced or scaled out won't inherit a category added by hand. Point the user to `spec.template.spec.additionalCategories` on the `NutanixMachineTemplate`.
- **ESXi VMs:** they use the `esxi_associateCategories`/`esxi_disassociateCategories` operations with the same pattern.
