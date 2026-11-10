---
name: nutanix-protection-policy
description: Create a Nutanix (Prism Central v4) protection policy — scheduled VM snapshots / backups targeted by category — through the self-hosted Nutanix MCP server, using a fixed lookup sequence and a known-good payload. Use this whenever the user asks to create, set up or add a backup policy, snapshot policy, protection policy, DR schedule or RPO for VMs in their Nutanix cluster, especially when they name a category (e.g. "ManagedBy:ZY") or an interval ("6-hourly", "every hour", "daily"), even if they don't say "protection policy".
---

# Nutanix protection policy (category-based VM snapshots)

Creates a `datapolicies` protection policy that snapshots every VM carrying a given
category on a schedule. Works through the Nutanix MCP server's `<namespace>_execute`
tools plus `getOperationSchema`.

## Inputs to pull from the request

| Input | Example | Default if unstated |
|---|---|---|
| Category `key:value` | `ManagedBy:ZY` | none — ask |
| Interval (RPO) | 6-hourly → `21600` s | none — ask |
| Retention | keep 4 | linear, enough points for ~24h (`86400 / RPO`, min 1) |
| Consistency | crash / app | `CRASH_CONSISTENT` (app-consistent needs NGT in guests) |
| Target | local / remote cluster | local only, on the AOS cluster |
| Policy name | | `pp-<key>-<value>-<interval>` lowercased, e.g. `pp-managedby-zy-6h` |

The user asking to create the policy is the go-ahead; don't stop to confirm defaults.
State every default you chose in the final reply so they can change it.

RPO conversions: hourly 3600, 2-hourly 7200, 4-hourly 14400, 6-hourly 21600,
12-hourly 43200, daily 86400.

## Step 0 — load the tools

The Nutanix MCP tools are deferred. Load them with `tool_search` before use:
`"nutanix datapolicies protection policy"`, `"nutanix prism clustermgmt"`,
and `"getOperationSchema"` (it does not come back with the namespace queries).

## Step 1 — lookups (independent; run them back to back)

1. **Category** — `prism_execute` `listCategories`
   `query_params: {"$filter": "key eq '<KEY>' and value eq '<VALUE>'", "$select": "extId,key,value"}`
   → keep `extId`. If nothing comes back, stop and tell the user the category does not
   exist; don't create it unasked.
2. **Cluster** — `clustermgmt_execute` `listClusters`
   `query_params: {"$select": "extId,name,config"}`
   → use clusters whose `config.clusterFunction` contains `AOS`. Ignore the
   `PRISM_CENTRAL` entry. One AOS cluster → use it. Several and the user didn't say →
   ask which.
3. **Domain manager (PC)** — `prism_execute` `listDomainManagers`
   `query_params: {"$select": "extId,config"}` → keep `extId`. In a single-PC setup this
   equals the PrismCentral cluster's extId.
4. **Existing policies** — `datapolicies_execute` `listProtectionPolicies`
   `query_params: {"$select": "extId,name,categoryIds"}` → if the category extId is
   already in another policy, stop and report that (a category can only belong to one
   protection policy). If a policy with the same name exists, pick a new name.
5. **Schema** — `getOperationSchema` `createProtectionPolicy`. Use it for field names,
   but **not** for `$objectType` values (see the gotcha below).

## Step 2 — create

`datapolicies_execute` `createProtectionPolicy`. Send only these fields; the
`NTNX-Request-Id` header is injected by the server.

```json
{
  "name": "<policy-name>",
  "description": "<interval> local snapshots for VMs with category <KEY>:<VALUE>",
  "categoryIds": ["<category-extId>"],
  "replicationLocations": [
    {
      "label": "local-<cluster-name-lowercase>",
      "domainManagerExtId": "<pc-extId>",
      "isPrimary": true,
      "replicationSubLocation": {
        "$objectType": "datapolicies.v4.config.NutanixCluster",
        "clusterExtIds": ["<aos-cluster-extId>"]
      }
    }
  ],
  "replicationConfigurations": [
    {
      "sourceLocationLabel": "local-<cluster-name-lowercase>",
      "schedule": {
        "recoveryPointObjectiveTimeSeconds": <RPO>,
        "recoveryPointType": "CRASH_CONSISTENT",
        "retention": {
          "$objectType": "datapolicies.v4.config.LinearRetention",
          "local": <N>
        }
      }
    }
  ]
}
```

### Gotcha: discriminator names

`getOperationSchema` reports discriminators as `datapolicies.v4.2.config.*`. Prism
Central (verified on pc.7.5.1.8) rejects those with a 400 `SchemaValidationError`.
Use `datapolicies.v4.config.*` (or `datapolicies.v4.r2.config.*`):

- `datapolicies.v4.config.NutanixCluster`
- `datapolicies.v4.config.Mst`
- `datapolicies.v4.config.LinearRetention`
- `datapolicies.v4.config.AutoRollupRetention`

If a 400 lists "possible values", take the accepted name from that message and retry once.

### Variants

- **Remote replication**: add a second `replicationLocations` entry (label e.g.
  `remote-<name>`, its own PC `domainManagerExtId` and cluster), set
  `remoteLocationLabel` on the configuration, and give the retention both `local` and
  `remote`. Add a reverse configuration (source = remote, remote = local) when the
  user wants failback protection.
- **Auto-rollup retention** (e.g. "keep 7 days"):
  `{"$objectType": "datapolicies.v4.config.AutoRollupRetention",
  "local": {"snapshotIntervalType": "DAILY", "frequency": 7}}` (frequency 1–24).
- **Application-consistent**: `"recoveryPointType": "APPLICATION_CONSISTENT"`. Mention
  that the guests need NGT.
- **Start time**: `"startTime": "HHh:MMm"` (e.g. `"02h:00m"`), only when the user asks for one.

## Step 3 — verify

The create returns a `TaskReference`. Call `prism_execute` `getTaskById` with
`path_params: {"extId": "<task extId>"}`. If the status is `RUNNING` or `QUEUED`, call it
again. On `SUCCEEDED`, the policy extId is in `completionDetails` under
`protectionPolicyExtId`. On `FAILED`, report `errorMessages`.

## Reply format

Keep it short:
- one line saying the policy was created and the task status;
- a small table with name and extId, category, location, schedule/RPO, consistency, retention;
- a sentence or two on the defaults you chose and how to change them. If the policy is
  local-only, say that it isn't off-cluster backup;
- if a discriminator retry was needed, mention it briefly.
