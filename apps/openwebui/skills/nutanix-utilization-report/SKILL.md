---
name: nutanix-utilization-report
description: Produce a utilization and health report for Nutanix clusters (compute, storage, network, alerts/events/tasks) through the Nutanix v4 MCP server, using a fixed minimal set of MCP calls and an embedded local analysis script instead of exploratory API calls. Use this whenever the user asks for a Nutanix / Prism Central / AOS / AHV cluster utilization report, capacity review, health check, N+1 or HA headroom check, "how are my clusters doing", or a periodic infra report — even if they don't say "utilization".
---

# Nutanix utilization report

Produces a RAG-scored report across four areas — compute, storage, network,
alerts/events/tasks — for every AOS cluster registered to Prism Central.

The design goal is **as few MCP calls as possible**. Every call in the plan
below has known-good parameters, so there is no need for `listOperations`,
`getOperationSchema`, or trial-and-error. Number crunching happens locally in
the embedded script (Appendix B), not through extra API calls.

## Call budget

**Core plan: 10 calls + 1 per AOS cluster** (+1 per extra page if more than
50 VMs). The previous hand-run version of this report took ~30 calls; most of
the savings come from:

- no schema/discovery calls (parameters are baked in below)
- one 7-day time series per *cluster*, never per host
- per-host CPU/memory and N+1 derived from the VM inventory + VM stats that are
  already fetched, instead of `getHostStats` per host
- one-bucket aggregates (`$samplingInterval=604800`) for VM stats instead of
  hourly series
- `$select` on every call that supports it, to keep payloads small

Do not add calls outside this plan unless the user asks for something it does
not cover (see "Optional calls").

## Step 0 — load tools (1 tool_search, not counted as MCP)

One `tool_search` with query `nutanix execute` and `limit: 10` loads
`clustermgmt_execute`, `vmm_execute`, `monitoring_execute`, `prism_execute`,
`networking_execute`. If one is missing, a second targeted search is fine.

## Step 1 — the call plan

Use window `END` = current hour (UTC, `YYYY-MM-DDTHH:00:00Z`) and
`START` = END − 7 days unless the user specifies another window.

| # | Tool / operation | Parameters | Feeds |
|---|---|---|---|
| 1 | `clustermgmt` `listClusters` | `$limit: 50` (no `$select` — nested selects are unreliable here) | cluster list; skip any whose `config.clusterFunction` contains `PRISM_CENTRAL` |
| 2..n | `clustermgmt` `getClusterStats` (one per AOS cluster) | path `extId`; `$startTime`, `$endTime`, `$samplingInterval: 3600`, `$statType: AVG`, `$select: hypervisorCpuUsagePpm,aggregateHypervisorMemoryUsagePpm,controllerAvgIoLatencyUsecs,controllerNumIops,ioBandwidthKbps,storageUsageBytes,storageCapacityBytes,logicalStorageUsageBytes,healthCheckScore,snapshotCapacityBytes,recycleBinUsageBytes` | cluster CPU/mem/latency/IOPS/capacity, peaks and when they happened |
| A | `clustermgmt` `listHosts` | `$limit: 100`, `$select: extId,hostName,cluster,numberOfCpuThreads,numberOfCpuCores,memorySizeBytes,maintenanceState,nodeStatus,hypervisor,blockModel` | host sizes |
| B | `vmm` `ahv_listVms` | `$limit: 100`, `$select: extId,name,numSockets,numCoresPerSocket,numThreadsPerCore,memorySizeBytes,powerState,host,cluster` | vCPU/RAM allocation per host, k8s control-plane placement |
| C | `vmm` `ahv_listVmStats` | `$startTime`, `$endTime`, `$samplingInterval: 604800`, `$statType: AVG`, `$limit: 50`, `$select: extId,stats/hypervisorCpuUsagePpm,stats/hypervisorCpuReadyTimePpm,stats/memoryUsagePpm` | per-VM CPU, CPU ready. **Server caps pages at 50** — add `$page: 1` etc. only if `totalAvailableResults` > 50 |
| D | `clustermgmt` `listDisks` | `$limit: 100` (**no `$select`** — it errors with CLU-11001) | disk health |
| E | `clustermgmt` `listHostNics` | `$limit: 100` | link state, speed, LLDP switch per uplink |
| F | `networking` `listVirtualSwitches` | none | bond mode, NIC membership, MTU |
| G | `monitoring` `listAlerts` | `$filter: isResolved eq false`, `$limit: 100`, `$orderby: creationTime desc`, `$select: extId,title,severity,alertType,sourceEntity/name,sourceEntity/type,creationTime,lastUpdatedTime,isAcknowledged,clusterUUID` | open alerts |
| H | `prism` `listTasks` | `$filter: createdTime gt <START> and status eq Prism.Config.TaskStatus'FAILED'`, `$limit: 100`, `$select: extId,operation,operationDescription,status,createdTime,entitiesAffected,errorMessages,ownedBy` | failed tasks |
| I | `monitoring` `listEvents` | `$filter: creationTime gt <START>`, `$limit: 100`, `$orderby: creationTime desc`, `$select: extId,message,eventType,sourceEntity/name,creationTime` (**`title` is not a valid field** for events) | HA failovers, reboots, user actions |

Issue independent calls back-to-back without pausing to analyse in between —
analysis happens once, at the end.

**If a call fails** with an OData/`$select` error, retry it **once** without
`$select`. Only call `getOperationSchema` if that retry also fails. If the MCP
server itself is unreachable (`server isn't responding`), stop and tell the
user — retrying in a loop just burns calls (a common cause is a stale MCP
session after the server restarted; reconnecting the connector fixes it).

## Optional calls (only when asked or clearly needed)

- **IPAM pool usage**: `networking` `listSubnets` (no `$select` — it errors),
  then `listVnicsBySubnetId` for the subnets that have a `poolList`. Count
  `assignedIpv4Addresses` inside the pool range.
- **Per-container usage**: `clustermgmt` `listStorageContainers` (`$select:
  containerExtId,name,maxCapacityBytes,replicationFactor,isCompressionEnabled,erasureCode,onDiskDedup`)
  — useful to flag compression off on the main VM container.
- **Exact per-host utilization** (instead of the VM-derived estimate):
  `getHostStats` per host with `$samplingInterval: 604800` — one small call
  per host. Use hourly sampling only if the user wants per-host peaks.

## Step 2 — analyse locally

This file is self-contained. The analysis script lives in **Appendix B** at the
end. Before the first run, write it to disk exactly as given (one bash call;
skip if `/home/claude/ntx/analyze.py` already exists):

```bash
mkdir -p /home/claude/ntx && cat > /home/claude/ntx/analyze.py <<'NTX_ANALYZE_EOF'
<paste the full contents of the Appendix B code block here, unchanged>
NTX_ANALYZE_EOF
```

The script needs only Python 3.8+ standard library and no network access.

Large results are auto-saved under `/mnt/user-data/tool_results/`. Small ones
come back inline — for those, write only the `payload.data` part to a file
under `/home/claude/ntx/` (or reason over them directly if tiny, e.g. a handful
of alerts). Then run:

```bash
python3 /home/claude/ntx/analyze.py \
  --cluster-stats <file>[,<file>...] \
  --hosts <file> --vms <file> --vmstats <file>[,<page2>] \
  --disks <file> --nics <file> --vswitches <file> \
  --alerts <file> --tasks <file> --events <file> \
  [--cvm-gib 32]
```

Every flag is optional; the script analyses whatever it is given and prints a
markdown block of findings plus a RAG verdict per area. It accepts raw tool
result files (with or without the `[{"text": ...}]` / `{"ok":…,"payload":…}`
wrappers). The docstring at the top of Appendix B lists what each section computes.

The script handles the parts that are tedious or error-prone by hand:
avg/p95/peak and the timestamps of the worst hours, per-host vCPU and RAM
allocation, the **N+1 memory check** (allocated VM memory of the largest host
vs. free allocatable memory on the rest), Kubernetes control-plane
co-location, single-uplink / single-switch detection, alert de-duplication by
type, and failed-task grouping (including the "same `If-Match` etag reused for
many VMs" client bug pattern).

## Step 3 — score and write

Thresholds are in **Appendix A** — use them when scoring. Overall area
status = worst status of its metrics. Apply judgment on top of the script:

- **Correlate spikes.** If the latency peak, a health-score dip, an alert burst
  and a failed task land in the same hour, report them as one incident.
- **Recurring daily peaks** (same hour every day) usually mean backups or a
  scheduled job — say so, and convert UTC to the user's local time.
- **Separate noise from signal.** Large counts of one alert type on
  non-certified / Community Edition hardware (e.g. "multiple disks mapped to
  single slot") are usually noise; say so, but don't let them hide the
  genuinely actionable ones (expiring passwords, CVM down, resiliency loss).
- **Failed tasks caused by user automation** (etag mismatches, test batches)
  are not infra problems; report them separately as client issues.
- **N+1 is about allocation, not usage**: AHV reserves a VM's full memory at
  power-on, so HA restart capacity depends on allocated RAM.

### Report structure

1. One line on scope (clusters, window, sampling).
2. **Scorecard table**: Area | Status | Main driver (one row per area).
3. One short section per area: key numbers first, then findings that matter.
   Use a small table only where it carries per-host or per-item comparisons.
4. **Recommended actions**, ordered by risk reduction (resiliency gaps before
   tidy-ups), each tied to a finding.

Keep it tight — the user wants verdicts and actions, not raw API output.
State where a number is derived or estimated (e.g. network throughput from
storage IO, since the v4 host-NIC API has no traffic counters).

If the user wants it as a shareable document, put the same structure in a doc.

---

## Appendix A — Scoring criteria

Default window: last 7 days, hourly samples. Report average and p95/peak.
An area's status is the worst status among its metrics.

### Compute

| Metric | Source | Green | Amber | Red |
|---|---|---|---|---|
| Cluster CPU | getClusterStats `hypervisorCpuUsagePpm` | avg < 60% | 60–80% | avg > 80% or sustained peak > 90% |
| Cluster memory | `aggregateHypervisorMemoryUsagePpm` | < 70% | 70–85% | > 85% |
| Per-host memory | VM allocation ÷ host RAM (or getHostStats) | < 70% | 70–85% | > 85% |
| N+1 (largest host fails) | allocated RAM of the failed host vs free allocatable RAM elsewhere | fits with > 10% spare | fits, < 10% spare | does not fit |
| VM CPU ready | `hypervisorCpuReadyTimePpm` (10 000 ppm = 1%) | < 2% | 2–5% | > 5% |
| vCPU : thread overcommit | VM inventory | ≤ 3:1 | 3–5:1 | > 5:1 |
| Host CPU imbalance (max − min) | per host | < 15 pts | 15–25 pts | > 25 pts |
| K8s control-plane spread | VM placement | all spread | — | quorum (≥2 of 3) on one host |

### Storage

| Metric | Source | Green | Amber | Red |
|---|---|---|---|---|
| Usage vs physical | `storageUsageBytes / storageCapacityBytes` | < 70% | 70–80% | > 80% |
| Usage vs resilient capacity | physical × (N−1)/N for RF2/FT1 | < 80% | 80–90% | > 90% |
| Controller latency (avg) | `controllerAvgIoLatencyUsecs` | < 3 ms | 3–10 ms | > 10 ms |
| Latency spikes | worst hourly sample | < 10 ms | 10–20 ms | > 20 ms (report as incident, area stays green if isolated) |
| Disk health | listDisks `status`, `diskAdvanceConfig.isOnline/isUnhealthy/isMarkedForRemoval` | all NORMAL/online | suspected unhealthy | offline / unhealthy |
| Hygiene | containers | — | compression off on main VM container | — |

### Network

| Metric | Source | Green | Amber | Red |
|---|---|---|---|---|
| Uplink redundancy | listHostNics `interfaceStatus` ("1" = up) per host | ≥ 2 up, ≥ 2 switches | ≥ 2 up, same switch | 1 up |
| Throughput vs link | estimated from cluster `ioBandwidthKbps` (×2 for RF2 writes) | < 50% | 50–70% | > 70% sustained |
| MTU consistency | vswitch / NICs + MTU alerts | consistent, no alerts | MTU alert open | — |
| IPAM pool usage (optional) | vNICs in pool range | < 70% | 70–85% | > 85% |

### Alerts / events / tasks

| Metric | Green | Amber | Red |
|---|---|---|---|
| Unresolved critical alerts | 0 | — | ≥ 1 actionable critical |
| Alerts unacknowledged > 24 h | 0 | any warnings | any critical |
| Failed tasks (window) | 0 | 1–5, or caused by user automation | > 5 infra failures, or repeated failure of the same infra op |
| Events | none notable | isolated HA failover / host reboot | recurring HA failovers on the same host |

---

## Appendix B — `analyze.py`

Write this verbatim to `/home/claude/ntx/analyze.py` (see Step 2).

```python
#!/usr/bin/env python3
"""Analyse Nutanix v4 API results for the utilization report. No network access.

Every input is optional; each section runs only if its inputs are given.
Inputs are raw tool-result files (any of these shapes are unwrapped):
  [{"text": "<json>"}]   {"ok":..., "payload": {"data": ...}}   {"data": ...}   [...]

Sections
  --cluster-stats  getClusterStats (hourly). avg / p95 / peak per metric, worst hours,
                   capacity %, resilient-capacity % (needs --nodes or hosts file).
  --hosts --vms --vmstats
                   per-host vCPU and RAM allocation, overcommit, VM-weighted CPU estimate,
                   N+1 memory check, CPU-ready top list, k8s control-plane co-location.
  --disks          disk health.
  --nics --vswitches
                   uplinks up per host, distinct switches, bond mode, MTU.
  --alerts         open alerts grouped by type, severity, ack state.
  --tasks          failed tasks grouped by operation/error; etag-reuse detection.
  --events         event types; HA failovers per host.
"""
import argparse, json, re, statistics as st, sys
from collections import Counter, defaultdict

GiB = 1024 ** 3


# ---------- loading ----------
def load(path):
    with open(path) as f:
        d = json.load(f)
    if isinstance(d, list) and d and isinstance(d[0], dict) and "text" in d[0] and len(d[0]) <= 2:
        d = json.loads(d[0]["text"])
    if isinstance(d, dict) and "payload" in d:
        d = d["payload"]
    if isinstance(d, dict) and "data" in d:
        d = d["data"]
    return d


def load_many(arg):
    out = []
    for p in (arg or "").split(","):
        if p.strip():
            d = load(p.strip())
            out.extend(d if isinstance(d, list) else [d])
    return out


def ref_id(x):
    if isinstance(x, dict):
        return x.get("extId") or x.get("uuid")
    return x


RAG = {0: "🟢 Green", 1: "🟡 Amber", 2: "🔴 Red"}


def rag(v, amber, red, reverse=False):
    if v is None:
        return 0
    if reverse:
        return 2 if v < red else 1 if v < amber else 0
    return 2 if v > red else 1 if v > amber else 0


def pct(x):
    return f"{x:.1f}%"


# ---------- cluster stats ----------
def series_summary(points):
    vals = [p["value"] for p in points if p.get("value") is not None]
    if not vals:
        return None
    s = sorted(vals)
    worst = sorted(points, key=lambda p: -(p.get("value") or 0))[:3]
    return dict(avg=st.mean(vals), p95=s[int(0.95 * (len(s) - 1))], max=s[-1], min=s[0],
                n=len(vals), first=points[-1]["value"], last=points[0]["value"],
                worst=[(p["timestamp"][5:16].replace("T", " "), p["value"]) for p in worst])


def cluster_section(files, nodes_by_cluster):
    lines, status, sstat = [], 0, 0
    for f in files:
        d = load(f)
        cid = d.get("extId", "?")
        S = {k: series_summary(v) for k, v in d.items() if isinstance(v, list) and v and isinstance(v[0], dict) and "value" in v[0]}
        lines.append(f"### Cluster {cid}")
        cpu, mem = S.get("hypervisorCpuUsagePpm"), S.get("aggregateHypervisorMemoryUsagePpm")
        if cpu:
            a, p, m = cpu["avg"] / 1e4, cpu["p95"] / 1e4, cpu["max"] / 1e4
            r = max(rag(a, 60, 80), 2 if m > 90 else 0); status = max(status, r)
            lines.append(f"- CPU: avg {pct(a)}, p95 {pct(p)}, peak {pct(m)} — {RAG[r]}; worst hours (UTC): " +
                         ", ".join(f"{t} {v/1e4:.0f}%" for t, v in cpu["worst"]))
            hours = Counter(t[-5:] for t, _ in cpu["worst"])
            if hours.most_common(1)[0][1] >= 2:
                lines.append(f"  - recurring peak at {hours.most_common(1)[0][0]} UTC — likely a scheduled job")
        if mem:
            a, p, m = mem["avg"] / 1e4, mem["p95"] / 1e4, mem["max"] / 1e4
            r = rag(a, 70, 85); status = max(status, r)
            lines.append(f"- Memory: avg {pct(a)}, p95 {pct(p)}, peak {pct(m)} — {RAG[r]}; peak at {mem['worst'][0][0]}")
        lat = S.get("controllerAvgIoLatencyUsecs")
        if lat:
            a, m = lat["avg"] / 1000, lat["max"] / 1000
            r = rag(a, 3, 10); sstat = max(sstat, r)
            lines.append(f"- Latency: avg {a:.2f} ms, p95 {lat['p95']/1000:.2f} ms, worst hour {m:.1f} ms at {lat['worst'][0][0]} — {RAG[r]}"
                         + (" (isolated spike, investigate)" if m > 20 and r == 0 else ""))
        for k, label, fmt in (("controllerNumIops", "IOPS", lambda v: f"{v:,.0f}"),
                              ("ioBandwidthKbps", "IO bandwidth", lambda v: f"{v/1024:.0f} MB/s")):
            if S.get(k):
                lines.append(f"- {label}: avg {fmt(S[k]['avg'])}, peak {fmt(S[k]['max'])} at {S[k]['worst'][0][0]}")
        if S.get("ioBandwidthKbps"):
            gbps = S["ioBandwidthKbps"]["max"] * 8 / 1e6
            lines.append(f"  - network estimate from storage IO: peak ≈ {gbps:.2f} Gbps ×2 for RF2 writes ≈ {2*gbps:.2f} Gbps")
        hs = S.get("healthCheckScore")
        if hs and hs["min"] < hs["max"]:
            lines.append(f"- Health score dipped to {hs['min']} at {sorted(d['healthCheckScore'], key=lambda p: p['value'])[0]['timestamp'][5:16]}")
        cap, used = S.get("storageCapacityBytes"), S.get("storageUsageBytes")
        if cap and used:
            u = used["last"] / cap["last"] * 100
            growth = (used["last"] - used["first"]) / 1e9
            r = rag(u, 70, 80); sstat = max(sstat, r)
            line = f"- Storage: {used['last']/1e12:.2f} TB of {cap['last']/1e12:.2f} TB ({pct(u)}), change over window {growth:+.0f} GB — {RAG[r]}"
            n = nodes_by_cluster.get(cid)
            if n and n > 1:
                res = cap["last"] * (n - 1) / n
                rr = rag(used["last"] / res * 100, 80, 90); sstat = max(sstat, rr)
                line += f"; vs resilient capacity ({n}-node, lose 1) {used['last']/res*100:.0f}%"
            lines.append(line)
        lines.append("")
    return status, sstat, lines


# ---------- capacity / placement ----------
CP_PATTERNS = [re.compile(r"^(?P<c>.+)-cp-\d+$"),
               re.compile(r"^(?P<c>.+?)-[a-z0-9]{5}-[a-z0-9]{5}$")]  # CAPI KubeadmControlPlane naming


def cp_group(name):
    if re.search(r"-md-|worker|-wn-", name):
        return None
    for p in CP_PATTERNS:
        m = p.match(name)
        if m:
            return m.group("c")
    return None


def capacity_section(hosts, vms, vmstats, cvm_gib):
    lines, status = [], 0
    H = {h["extId"]: h for h in hosts}
    stats = {}
    for s in vmstats:
        tup = [t for t in s.get("stats", []) if t.get("hypervisorCpuUsagePpm") is not None]
        if tup:
            stats[s["extId"]] = {k: st.mean(t.get(k, 0) or 0 for t in tup) for k in
                                 ("hypervisorCpuUsagePpm", "hypervisorCpuReadyTimePpm", "memoryUsagePpm")}
    per = defaultdict(lambda: dict(vms=0, vcpu=0, mem=0, cpu_threads_used=0.0))
    total_vcpu, cp = 0, defaultdict(Counter)
    for v in vms:
        if v.get("powerState") != "ON":
            continue
        hid = ref_id(v.get("host"))
        vcpu = (v.get("numSockets") or 1) * (v.get("numCoresPerSocket") or 1) * (v.get("numThreadsPerCore") or 1)
        p = per[hid]; p["vms"] += 1; p["vcpu"] += vcpu; p["mem"] += v.get("memorySizeBytes") or 0
        total_vcpu += vcpu
        if v["extId"] in stats:
            p["cpu_threads_used"] += stats[v["extId"]]["hypervisorCpuUsagePpm"] / 1e6 * vcpu
        g = cp_group(v.get("name", ""))
        if g:
            cp[g][H.get(hid, {}).get("hostName", hid)] += 1

    rows, cpu_est = [], []
    lines.append("| Host | RAM | VMs | vCPU:thread | VM RAM alloc | est. CPU from VMs |")
    lines.append("|---|---|---|---|---|---|")
    for hid, h in H.items():
        p = per.get(hid, dict(vms=0, vcpu=0, mem=0, cpu_threads_used=0))
        ram = h.get("memorySizeBytes") or 0
        thr = h.get("numberOfCpuThreads") or 1
        alloc = (p["mem"] + cvm_gib * GiB) / ram * 100 if ram else 0
        est = p["cpu_threads_used"] / thr * 100
        cpu_est.append(est)
        rows.append((hid, h, ram, p))
        r = rag(alloc, 70, 85); status = max(status, r)
        oc = p["vcpu"] / thr
        status = max(status, rag(oc, 3, 5))
        lines.append(f"| {h.get('hostName', hid)} | {ram/GiB:,.0f} GiB | {p['vms']} | {oc:.2f}:1 | {pct(alloc)} {RAG[r][:1]} | ~{est:.0f}% |")
    threads = sum((h.get("numberOfCpuThreads") or 0) for h in H.values())
    lines.append(f"\n- Cluster vCPU {total_vcpu} on {threads} threads = {total_vcpu/max(threads,1):.2f}:1")
    on = [v for v in vms if v.get("powerState") == "ON"]
    coverage = sum(1 for v in on if v["extId"] in stats) / max(len(on), 1)
    if coverage < 0.8:
        lines.append(f"- VM stats cover only {coverage:.0%} of powered-on VMs — CPU estimate/imbalance skipped (fetch remaining pages)")
    elif len(cpu_est) > 1:
        spread = max(cpu_est) - min(cpu_est)
        r = rag(spread, 15, 25); status = max(status, r)
        lines.append(f"- Host CPU imbalance (VM-derived, excludes CVM): {spread:.0f} pts — {RAG[r]}")

    # N+1: fail each host, can the rest absorb its allocated memory?
    if len(rows) > 1:
        lines.append(f"- N+1 memory (allocation-based, CVM reserve {cvm_gib} GiB/host):")
        for hid, h, ram, p in sorted(rows, key=lambda r: -r[2]):
            need = p["mem"]
            free = sum(max(r2 - (p2["mem"] + cvm_gib * GiB), 0) for hid2, _, r2, p2 in rows if hid2 != hid)
            if need <= free * 0.9:
                r = 0
            elif need <= free:
                r = 1
            else:
                r = 2
            status = max(status, r)
            lines.append(f"  - lose {h.get('hostName')}: {need/GiB:,.0f} GiB to restart vs {free/GiB:,.0f} GiB free — {RAG[r]}"
                         + (f" (short {(need-free)/GiB:,.0f} GiB)" if r == 2 else ""))

    if stats:
        names = {v["extId"]: v.get("name") for v in vms}
        top = sorted(stats.items(), key=lambda kv: -kv[1]["hypervisorCpuReadyTimePpm"])[:3]
        worst = top[0][1]["hypervisorCpuReadyTimePpm"] / 1e4
        r = rag(worst, 2, 5); status = max(status, r)
        lines.append(f"- CPU ready worst VMs: " + ", ".join(f"{names.get(k, k)} {s['hypervisorCpuReadyTimePpm']/1e4:.2f}%" for k, s in top) + f" — {RAG[r]}")
        busy = sorted(stats.items(), key=lambda kv: -kv[1]["hypervisorCpuUsagePpm"])[:5]
        lines.append("- Busiest VMs: " + ", ".join(f"{names.get(k, k[:8])} {s['hypervisorCpuUsagePpm']/1e4:.0f}%" for k, s in busy))

    if cp:
        lines.append("- K8s control-plane placement (name-based heuristic — verify):")
        for g, hosts_c in sorted(cp.items()):
            n = sum(hosts_c.values())
            bad = n >= 2 and max(hosts_c.values()) > n // 2
            if bad:
                status = 2
            lines.append(f"  - {g}: " + ", ".join(f"{h}×{c}" for h, c in hosts_c.items()) + (" — 🔴 quorum on one host" if bad else " — ok"))
    return status, lines


# ---------- disks ----------
def disk_section(disks):
    bad = [d for d in disks if d.get("status") != "NORMAL" or not d.get("diskAdvanceConfig", {}).get("isOnline", True)
           or d.get("diskAdvanceConfig", {}).get("isUnhealthy") or d.get("diskAdvanceConfig", {}).get("isMarkedForRemoval")]
    sus = [d for d in disks if d.get("diskAdvanceConfig", {}).get("isSuspectedUnhealthy")]
    by = Counter((d.get("hostName"), d.get("storageTier"), d.get("model")) for d in disks)
    status = 2 if bad else 1 if sus else 0
    lines = [f"- Disks: {len(disks)} total, {len(bad)} unhealthy/offline, {len(sus)} suspected — {RAG[status]}"]
    models = Counter((d.get("storageTier"), d.get("model")) for d in disks)
    lines.append("  - mix: " + ", ".join(f"{n}× {m} ({t})" for (t, m), n in models.items()))
    for d in bad:
        lines.append(f"  - {d.get('hostName')} {d.get('serialNumber')} status={d.get('status')}")
    return status, lines


# ---------- network ----------
def net_section(nics, vswitches, hosts):
    names = {h["extId"]: h.get("hostName") for h in hosts}
    per = defaultdict(list)
    for n in nics:
        per[n.get("nodeUuid")].append(n)
    status, lines = 0, []
    all_switches = set()
    for hid, ns in per.items():
        up = [n for n in ns if str(n.get("interfaceStatus")) == "1"]
        sw = {n.get("switchManagementIp", {}).get("ipv4", {}).get("value") or n.get("switchMacAddress") for n in up}
        sw.discard(None); all_switches |= sw
        r = 2 if len(up) < 2 else 1 if len(sw) < 2 else 0
        status = max(status, r)
        up_desc = ", ".join("%s %dG" % (n.get("name"), (n.get("linkSpeedInKbps") or 0) // 1000000) for n in up)
        lines.append(f"- {names.get(hid, hid)}: {len(up)}/{len(ns)} NICs up ({up_desc}), "
                     f"switches {sorted(sw) or '?'} — {RAG[r]}")
    if per and len(all_switches) == 1:
        lines.append(f"- All active uplinks land on one switch ({next(iter(all_switches))}) — cluster-wide single point of failure")
    for vs in vswitches:
        lines.append(f"- vSwitch {vs.get('name')}: bond {vs.get('bondMode')}, MTU {vs.get('mtu')}")
    return status, lines


# ---------- alerts / tasks / events ----------
def alert_section(alerts):
    g = defaultdict(lambda: dict(n=0, hosts=set(), first=None, last=None, ack=0))
    for a in alerts:
        k = (a.get("severity"), a.get("alertType"), a.get("title"))
        x = g[k]; x["n"] += 1; x["hosts"].add((a.get("sourceEntity") or {}).get("name"))
        t = a.get("creationTime", "")[:10]
        x["first"] = min(filter(None, [x["first"], t])); x["last"] = max(filter(None, [x["last"], t]))
        x["ack"] += bool(a.get("isAcknowledged"))
    crit = sum(x["n"] for k, x in g.items() if k[0] == "CRITICAL")
    unack = sum(x["n"] - x["ack"] for x in g.values())
    status = 2 if crit else 1 if g else 0
    lines = [f"- {len(alerts)} unresolved ({crit} critical), {unack} unacknowledged — {RAG[status]}"]
    order = {"CRITICAL": 0, "WARNING": 1, "INFO": 2}
    for (sev, typ, title), x in sorted(g.items(), key=lambda kv: (order.get(kv[0][0], 3), -kv[1]["n"])):
        lines.append(f"  - {x['n']}× {sev} {typ} \"{title}\" on {', '.join(sorted(filter(None, x['hosts'])))} ({x['first']} → {x['last']})")
    return status, lines


def task_section(tasks):
    g = defaultdict(list)
    for t in tasks:
        errs = t.get("errorMessages") or [{}]
        g[(t.get("operation"), errs[0].get("errorGroup") or errs[0].get("code"))].append(t)
    lines = [f"- {len(tasks)} failed tasks in window"]
    infra_fail = 0
    for (op, err), ts in sorted(g.items(), key=lambda kv: -len(kv[1])):
        owners = {(t.get("ownedBy") or {}).get("name") for t in ts} - {None}
        lines.append(f"  - {len(ts)}× {op} [{err}] owners {sorted(owners) or ['system']}")
        etags = Counter((e.get("argumentsMap") or {}).get("request_etag") for t in ts for e in (t.get("errorMessages") or []))
        etags.pop(None, None)
        if etags and etags.most_common(1)[0][1] > 1:
            tag, n = etags.most_common(1)[0]
            lines.append(f"    - same If-Match '{tag}' sent for {n} different entities → client reuses one etag (automation bug, not infra)")
        elif not owners:
            infra_fail += len(ts)
    status = 2 if infra_fail > 5 else 1 if tasks else 0
    lines[0] += f" — {RAG[status]}"
    return status, lines


def event_section(events, hosts):
    names = {h["extId"]: h.get("hostName") for h in hosts}
    types = Counter(e.get("eventType") for e in events)
    lines = [f"- {len(events)} events in window: " + (", ".join(f"{t}×{n}" for t, n in types.most_common(8)) or "none")]
    ha = Counter()
    for e in events:
        if "HAFailover" in (e.get("eventType") or ""):
            node = next((p["paramValue"].get("stringValue") for p in e.get("parameters", []) if p.get("paramName") == "node_uuid"), None)
            ha[names.get(node, node or "?")] += 1
    status = 2 if any(n > 1 for n in ha.values()) else 1 if ha else 0
    for h, n in ha.items():
        lines.append(f"  - HA failover from {h}: {n}×")
    return status, lines


def main():
    ap = argparse.ArgumentParser()
    for a in ("cluster-stats", "hosts", "vms", "vmstats", "disks", "nics", "vswitches", "alerts", "tasks", "events"):
        ap.add_argument("--" + a)
    ap.add_argument("--cvm-gib", type=float, default=32, help="RAM reserved per host for the CVM (default 32)")
    ap.add_argument("--nodes", help="cluster_extId=N[,..] if hosts file not given")
    a = ap.parse_args()

    hosts = load_many(a.hosts)
    nodes_by = dict(Counter(ref_id(h.get("cluster")) for h in hosts if h.get("cluster")))
    if a.nodes:
        nodes_by.update({k: int(v) for k, v in (x.split("=") for x in a.nodes.split(","))})
    if not nodes_by and hosts and a.cluster_stats and "," not in a.cluster_stats:
        nodes_by[load(a.cluster_stats).get("extId")] = len(hosts)  # single cluster: all hosts belong to it

    out, verdict = [], {}

    def sec(area, st_lines):
        s, l = st_lines
        verdict[area] = max(verdict.get(area, 0), s)
        out.extend(l)

    if a.cluster_stats:
        out.append("## Cluster time series")
        c, sst, l = cluster_section(a.cluster_stats.split(","), nodes_by)
        sec("Compute", (c, l)); verdict["Storage"] = max(verdict.get("Storage", 0), sst)
    if hosts and a.vms:
        out.append("## Compute — hosts & placement")
        sec("Compute", capacity_section(hosts, load_many(a.vms), load_many(a.vmstats), a.cvm_gib))
    if a.disks:
        out.append("## Storage — disks"); sec("Storage", disk_section(load_many(a.disks)))
    if a.nics:
        out.append("## Network"); sec("Network", net_section(load_many(a.nics), load_many(a.vswitches), hosts))
    if a.alerts:
        out.append("## Alerts"); sec("Alerts/events/tasks", alert_section(load_many(a.alerts)))
    if a.tasks:
        out.append("## Failed tasks"); sec("Alerts/events/tasks", task_section(load_many(a.tasks)))
    if a.events is not None:
        out.append("## Events"); sec("Alerts/events/tasks", event_section(load_many(a.events), hosts))

    print("## Script verdict (before judgment)")
    for k, v in verdict.items():
        print(f"- {k}: {RAG[v]}")
    print()
    print("\n".join(out))


if __name__ == "__main__":
    main()
```
