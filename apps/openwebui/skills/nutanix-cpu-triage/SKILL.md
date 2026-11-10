---
name: nutanix-cpu-triage
description: Triage CPU questions on Nutanix AHV clusters through the Nutanix MCP server (Prism Central v4 APIs). Gets cluster and host CPU, ranks the top VMs, and correlates CPU alerts and recent Prism tasks to explain what is driving load. Use this whenever the user asks about CPU utilization, CPU alerts, "high CPU", busy or hot hosts, noisy-neighbour VMs, top CPU consumers, CPU ready or contention, or wants to know why a Nutanix cluster, host or VM is slow or alerting. This applies even if they only say "cluster is alerting", "what's eating CPU" or "check the cluster", and whether or not they name Prism or the MCP server.
---

# Nutanix CPU Triage

Answer "how is CPU doing and why" for Nutanix AHV clusters. Use live data from the Nutanix MCP server, whose tools are named `mcp__nutanix-mcp__...__<namespace>_execute`. Load them with `tool_search` if they are deferred.

The goal is a short, evidence-based verdict:
- Is CPU actually a problem right now?
- If so, where is the load: the cluster, a single host, or a few VMs?
- What alert or task explains it?

Everything here is read-only. Never acknowledge or resolve alerts, migrate VMs, change power state, or edit policies unless the user explicitly asks in chat. In that case, confirm the exact action first.

Before your first call to any operation, call `getOperationSchema` for it. Parameter names and filterable fields differ between operations. The **API cheatsheet** at the end of this file has the shapes verified against this server and the gotchas found along the way. Read it before your first call.

## Workflow

Scale the depth to the question. For "top 5 VMs by CPU", steps 1, 2 and 4 are enough. For "I'm getting CPU alerts" or "why is it slow", run the full workflow.

### 0. Establish "now"

The stats APIs reject any `$startTime`/`$endTime` in the future with error `CLU-10006`. Get the real UTC time with `date -u +%Y-%m-%dT%H:%M:%SZ` in bash, and end windows 1–2 minutes before it. If bash isn't available and you get CLU-10006, step the end time back and retry. Never guess a time window ahead of the clock.

### 1. Scope the clusters

Call `clustermgmt listClusters` with `$select=extId,name,config`. Skip any entry whose `config.clusterFunction` contains `PRISM_CENTRAL`. That is the PC itself, not a workload cluster. If there are several AOS clusters and the user didn't name one, cover all of them briefly.

### 2. Cluster CPU trend

Call `clustermgmt getClusterStats` with:
- a 2-hour window
- `$samplingInterval=300`
- `$statType=AVG`
- `$select=hypervisorCpuUsagePpm,aggregateHypervisorMemoryUsagePpm`

Report the latest value, the min–max range, and whether it is flat, rising or spiking. If the user mentioned a specific alert time, widen the window to cover it.

### 3. Per-host CPU (do this for any alert or "slow" question)

1. Call `clustermgmt listHostsByClusterId` with `$select=extId,hostName,numberOfCpuCores,numberOfCpuThreads,cpuModel`.
2. For each host, call `clustermgmt getHostStats` with path params `clusterExtId` and `extId`, a 1-hour window, and `$samplingInterval=900`.

Look for imbalance between hosts. A cluster average hides a hot host, and host-scoped alerts fire on the hot host, not the average. A spread of more than about 20 points between the busiest and the idlest host is worth calling out.

### 4. Top VMs

1. Call `vmm ahv_listVmStats` with:
   - a 15-minute window
   - `$samplingInterval=900`
   - `$statType=AVG`
   - `$orderby=stats/hypervisorCpuUsagePpm desc`
   - `$select=extId,stats/hypervisorCpuUsagePpm,stats/hypervisorCpuReadyTimePpm,stats/numVcpusUsedPpm`

   Page through the results with `$page`. The server caps each page at 50 even if you ask for more.
2. Call `vmm ahv_listVms` with `$select=extId,name,numSockets,numCoresPerSocket,numThreadsPerCore,host,powerState` to get names, vCPU counts and current host.
3. Join the two on `extId`.

Rank by **absolute vCPUs used** (`numVcpusUsedPpm / 1e6`). That number shows what actually loads a host. Show "% of its own allocation" as a second column. A 4-vCPU VM at 33% uses less host CPU than a 16-vCPU VM at 13%. If the user asks for "busiest relative to size", rank by percent instead. When the two rankings differ meaningfully, mention it in one line.

VMs that appear in stats but not in `ahv_listVms` are almost always **CVMs**, one per host. Label them as CVMs, list them separately from user VMs, and treat 20–30% CVM usage as normal storage-stack overhead. The Prism Central VM does appear by name (e.g. `NTNX-...-PCVM-...`); call it out as PC.

Flag CPU ready above about 5% (50,000 ppm) on any VM. That means the VM is waiting for physical cores, which is real contention, even if utilization looks moderate. Ready under 2% means there is no contention.

### 5. Alerts

Call `monitoring listAlerts` with:
- `$filter` on `creationTime gt <window start>`, optionally combined with `isResolved eq false`
- `$orderby=creationTime desc`
- `$select=extId,title,message,parameters,severity,alertType,classifications,sourceEntity/name,sourceEntity/type,creationTime,lastUpdatedTime,isResolved,isAcknowledged,metricDetails`

`title` is not filterable server-side. Pull the window and match CPU alerts client-side: look for "cpu" (case-insensitive) in `title`, `message`, or `metricDetails`.

Titles and messages contain `{placeholders}`. Fill them from `parameters` when you quote an alert.

If no CPU alert exists in Prism Central for the window, say so plainly. Then point out the likely alternatives:
- The alert came from somewhere else (an external monitoring stack, or a Prism Element-only alert).
- A user-defined policy fired and auto-resolved. Check with `monitoring listUdaPolicies` to see whether any custom CPU thresholds exist.

Ask the user for the alert's time or text if that would settle it.

When an alert is found, match its `sourceEntity` to the host or VM data from steps 3 and 4. Re-pull stats for the alert's creation time if it is older than your current window.

### 6. Tasks (correlation)

Call `prism listTasks` with:
- `$filter=createdTime gt <alert time − 30m>` (bounded by the alert time + 30m if the alert is old)
- `$orderby=createdTime desc`
- `$select=extId,operation,operationDescription,status,progressPercentage,createdTime,completedTime,entitiesAffected,ownedBy/name`

Look for operations that plausibly cause CPU load:
- power-ons and VM creates or clones
- migrations or host maintenance (load shifts to the remaining hosts)
- snapshots and recovery points
- LCM or upgrade tasks
- image imports
- bulk batches

Treat category and metadata updates as noise unless nothing else turns up. Name the task, who ran it, and when. Only claim a causal link when the timing and the affected entity actually line up.

### 7. Placement sanity check (when a host is hot)

Using the `host` field from `ahv_listVms`, count VMs per host. Check whether replicas of the same control plane share one host. Typical patterns are names with `-cp-N`, or NKP control-plane nodes like `<cluster>-<5char>-<5char>` that have no `-md-` in the name. Sharing a host is both a load-concentration issue and a single-host-failure risk. Suggest VM-VM anti-affinity policies, and ADS if it appears disabled. Don't create the policies unless the user asks.

## Interpreting units

- `*Ppm` values are parts per million: divide by 10,000 to get a percentage.
- Host and cluster `hypervisorCpuUsagePpm` is a share of the host or cluster's physical CPU.
- VM `hypervisorCpuUsagePpm` is a share of **that VM's own vCPUs**, not of the host.
- `numVcpusUsedPpm / 1e6` is the number of vCPUs actively in use.
- Stats responses sometimes return two tuples for one sampling window. Average them.

## Output format

Lead with the verdict in one or two sentences, e.g. "CPU is fine at 43% and flat; the load is concentrated on host X." Then use this structure and omit sections the question didn't need:

```
**Cluster <name>** — <latest %> (range <a–b%> over <window>), memory <%>, ready time <ok/elevated>

| Host | CPU |                       ← only if host data was pulled
| VM | vCPUs used | % of allocated | ← top N user VMs; CVMs noted in one line below

**Alerts:** <CPU alerts found, with entity and time — or "none in PC for <window>" plus next step>
**Correlated tasks:** <only tasks that plausibly explain the load>
**Takeaway / next step:** <1–3 sentences: cause, and the one or two actions worth taking>
```

Keep it tight. The user is an engineer and wants numbers and a conclusion, not a narration of the API calls. Include the timestamp (UTC) the data is as of. If a call fails or returns partial data (`isTruncated`, missing pages), say what is missing instead of silently ranking an incomplete set.

---

## API cheatsheet

Every operation below was verified against the self-hosted Nutanix MCP server, which fronts the Prism Central v4.2/v4.3 APIs. Still call `getOperationSchema` before the first use of an operation in a session, because server versions drift.

Calling convention: `<namespace>_execute(operation=..., path_params={...}, query_params={...})`. Query param keys keep their `$` prefix.

### 1. clustermgmt

| Operation | Key params | Notes |
|---|---|---|
| `listClusters` | `$select=extId,name,config` | Skip entries whose `config.clusterFunction` contains `PRISM_CENTRAL`. |
| `getClusterStats` | path `extId`; query `$startTime`, `$endTime` (required), `$samplingInterval`, `$statType=AVG`, `$select` | Useful fields: `hypervisorCpuUsagePpm`, `aggregateHypervisorMemoryUsagePpm`. `cpuUsageHz`/`cpuCapacityHz` may come back empty. Returns arrays of `{timestamp, value}`, newest first. |
| `listHostsByClusterId` | path `clusterExtId`; `$select=extId,hostName,numberOfCpuCores,numberOfCpuThreads,cpuModel` | |
| `getHostStats` | path `clusterExtId` **and** `extId`; same time params | `$select=hypervisorCpuUsagePpm`. |

### 2. vmm

| Operation | Key params | Notes |
|---|---|---|
| `ahv_listVmStats` | `$startTime`, `$endTime` (required), `$samplingInterval`, `$statType`, `$orderby=stats/hypervisorCpuUsagePpm desc`, `$select=extId,stats/hypervisorCpuUsagePpm,stats/hypervisorCpuReadyTimePpm,stats/numVcpusUsedPpm`, `$page`, `$limit` | Stat fields in `$select`, `$filter` and `$orderby` need the `stats/` prefix. The page size is effectively capped at 50, so follow `metadata.links` rel `next` / `totalAvailableResults`. Includes CVMs. |
| `ahv_getVmStatsById` | path `extId` | Use for a time series on one VM (e.g. around an alert time). |
| `ahv_listVms` | `$select=extId,name,numSockets,numCoresPerSocket,numThreadsPerCore,host,powerState`, `$limit` up to 100 | vCPUs = sockets × cores × threads. `host.extId` gives current placement. Excludes CVMs. `metadata.extraInfo.truncatedIndexes` can list entries with partially returned fields. The names are still usable. |

Other useful stat fields: `stats/hypervisorMemoryUsagePpm`, `stats/guestMemoryUsagePpm`, `stats/hypervisorSwapInRateKbps`. Swap activity sometimes shows up as perceived "CPU" slowness.

### 3. monitoring

| Operation | Key params | Notes |
|---|---|---|
| `listAlerts` | `$filter` (filterable: `creationTime`, `lastUpdatedTime`, `severity`, `isResolved`, `isAcknowledged`, `alertType`, `classifications`, `impactTypes`, `sourceEntity/name`, `sourceEntity/type`, `sourceEntity/extId`, `clusterName`), `$orderby=creationTime desc`, `$select` (adds `title`, `message`, `parameters`, `metricDetails`, `affectedEntities`, `rootCauseAnalysis`, `kbArticles`) | `title`/`message` are **not** filterable, so match "cpu" client-side. Example filter: `creationTime gt 2026-10-01T00:00:00Z and isResolved eq false`. `sourceEntity.type` values seen: `node`, `cluster`, `vm`. |
| `getAlertById` | path `extId` | Full detail, including `metricDetails` and RCA. |
| `listEvents` | similar to alerts | Lower-severity signals that may not have raised an alert. |
| `listUdaPolicies` | — | User-defined alert policies. This is where custom CPU thresholds live. |
| `manageAlert` | — | **Write.** Acknowledge or resolve. Only on explicit user request, with confirmation. |

### 4. prism

| Operation | Key params | Notes |
|---|---|---|
| `listTasks` | `$filter` (`createdTime`, `completedTime`, `status`, `operation`, `entitiesAffected/name`, `entitiesAffected/extId`, `clusterExtIds`, `isBackgroundTask`), `$orderby=createdTime desc`, `$select=extId,operation,operationDescription,status,progressPercentage,createdTime,completedTime,entitiesAffected,ownedBy/name,errorMessages` | Task `extId`s look like `ZXJnb24=:<uuid>`; pass them as-is. `entitiesAffected[].rel` is like `vmm:ahv:config:vm`. |
| `getTaskById` | path `extId` | Includes subtasks and errors. |

CPU-relevant task operations: VM power on/off, create/clone/deploy, migrate (including cross-cluster), host maintenance enter/exit, recovery point/snapshot creation, LCM inventory/update, image import, batch submits. Low signal: category associate/disassociate, metadata updates.

### 5. Gotchas

- **Future timestamps** fail with `CLU-10006 ... refer to time in future`. Anchor on `date -u`.
- **Sampling**: with `$samplingInterval` equal to the window length you usually get one value, but VM stats can return two tuples. Average them.
- **Units**: ppm ÷ 10,000 = %. A VM's `hypervisorCpuUsagePpm` is relative to its own vCPUs. `numVcpusUsedPpm` ÷ 1e6 = vCPUs in use.
- **CVM identification**: present in `ahv_listVmStats`, absent from `ahv_listVms`. There is one per host, usually 8–12 vCPUs.
- **Two cluster objects**: PC registers itself as a cluster (`clusterFunction: PRISM_CENTRAL`). Alerts about PC may show `sourceEntity.name` = the PC cluster name.
- **Stale views**: alert `lastUpdatedTime` refreshes on re-evaluation, so use `creationTime` for "when did it start".
