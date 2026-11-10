---
name: nutanix-vm-list
description: List Nutanix virtual machines and organize them into meaningful groups inferred from the fleet itself. Use this skill whenever the user asks for a VM list, VM inventory, what VMs exist, what is running on their Nutanix cluster or Prism Central, or asks to see VMs grouped, sorted, or made sense of — even if they never say "Nutanix" or "Prism" explicitly.
---

# Nutanix VM List

List VMs from Prism Central via the Nutanix MCP server, then organize them into
groups you derive from the data — not from a fixed taxonomy.

## Read-only

Use only `ahv_listVms` and `ahv_getVmById` on `vmm_execute`. That tool also
fronts power, delete, and reconfigure operations — never pass one while running
this skill. If the user wants an action taken, show the list first and ask.

## Step 1 — List the VMs

```
vmm_execute(operation="ahv_listVms", $limit=100, $page=0)
```

Query parameters go as top-level arguments alongside `operation`, not inside
`request_body`; omit `request_body` entirely for reads. Paginate until a page
returns fewer than `$limit`.

Keep per VM: `name`, `description`, `powerState`, `numSockets` ×
`numCoresPerSocket` (total vCPU), `memorySizeBytes`, `createTime`, and the
cluster reference.

If nothing comes back, the cluster may be ESXi rather than AHV — retry with
`esxi_listVms` before reporting an empty fleet.

This is the only call. Do not fetch per-VM detail unless the user asks about a
specific VM.

## Step 2 — Derive the groups

Read the whole list first, then decide how it naturally divides. Do not pick
groups before seeing the data — the point is to fit the scheme to this fleet.

Signals to group on, in rough order of usefulness:

- **Name structure.** Shared prefixes, suffixes, and numbering almost always
  encode purpose: `nkp-mgmt-cp-0` and `nkp-mgmt-md-2` belong together; `win-`,
  `db-`, `test-` mark themselves. Numbered siblings are one group, not several.
- **Sizing shape.** VMs with identical vCPU and memory are usually one pool —
  control planes, worker nodes, replicas.
- **Power state.** A cluster of powered-off VMs is its own story regardless of
  what they are named.
- **Age.** A batch created within minutes of each other was provisioned
  together.
- **Cluster**, when more than one is registered.

Aim for 3–7 groups. Every VM lands in exactly one; leftovers go in `Other`
rather than being forced somewhere. Name each group for what it appears to be
in plain language — "Kubernetes management cluster", "Windows test VMs",
"Powered-off, likely abandoned" — not "Group A".

## Step 3 — Report

```markdown
## <Group name> (N VMs)
Grouped by: <the evidence — shared prefix, identical sizing, both>

| VM | State | vCPU | Mem GiB |
```

Then two or three sentences: total count, how many powered off, and anything
that stood out — a VM that resisted grouping, a pool where one member is sized
differently from its siblings, an obvious naming inconsistency.

## Be honest that this is inference

The groups are read off names and shapes, not off anything authoritative. A VM
named `prod-db-01` might be an abandoned test box; identical sizing might be
coincidence. Say what each grouping rests on so the user can tell where you are
confident and where you are pattern-matching.

If names are opaque — UUIDs, or no shared structure — say the fleet has no
usable naming convention and fall back to grouping by size and power state.
That is a more useful answer than confident groups invented out of nothing.

If the user asked for a flat list instead, return a single table sorted by name.

## Gotchas

- The server asks callers to run `getOperationSchema` before executing. If a
  parameter is rejected, call it for that operation rather than guessing.
- If an operation id is rejected outright, find the current name with
  `listOperations(namespace="vmm", search="list vms")`.
- On a 403, call `getOperationPermissions` and report the missing role instead
  of retrying.
