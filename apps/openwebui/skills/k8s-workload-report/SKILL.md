---
name: k8s-workload-report
description: Audit Kubernetes workloads and produce a health report that highlights failures, degraded workloads, resource pressure, and configuration smells. Use this skill whenever the user asks about cluster health, workload status, whether anything is broken or needs attention, what changed in a cluster, a cluster review, audit, or sanity check, or asks to look over pods, nodes, or namespaces — even if they never say the word "report". Also use it when the user asks a narrow diagnostic question about a specific workload, since the same collection and triage logic applies at smaller scope.
---

# Kubernetes Workload Health Report

Inspect a cluster's workloads and report what is broken, what is degraded, and
what is merely unusual — with evidence for every claim.

## Read-only contract

Do not create, patch, scale, restart, delete, or exec into anything, and do not
cordon or drain nodes. A health check that changes cluster state is no longer a
health check: it destroys the evidence it was supposed to gather and makes the
report unreproducible. Fixes belong in "Recommended actions" as commands the
user runs themselves.

The one exception is if the user explicitly asks for a fix in the same breath
("check the cluster and restart anything crashlooping"). Even then, report
first, then ask before acting.

## Step 1 — Establish scope

Settle three things before collecting. Infer them from the request when clear;
ask only if genuinely ambiguous.

- **Which cluster**, if more than one context is reachable.
- **Which namespaces** — all by default, excluding nothing. System namespaces
  often hold the actual problem.
- **Depth**:
  - **Triage** (default for narrow questions like "why is X down"): collect only
    what bears on the named workload. Skip the inventory table. Answer in prose.
  - **Full report** (default for "how does the cluster look"): everything below.

## Step 2 — Collect

Use the Kubernetes tools available — an MCP connector if present, otherwise
`kubectl`. Either way the sequence is broad first:

1. List namespaces to fix scope.
2. List pods across scope — the core inventory.
3. List workload controllers: Deployments, StatefulSets, DaemonSets, Jobs,
   CronJobs, plus PVCs. Needed to compare desired against ready.
4. List events, filtered to warnings where possible.
5. Pull node and pod resource consumption. If the metrics API is unavailable,
   say so in the report rather than silently omitting the resource section.
6. **Only for workloads already flagged suspicious**: fetch logs (tail, plus the
   previous container if it restarted) and the full resource spec.

Step 6 is where budgets are blown. Do not fetch logs for healthy pods. Cap the
run at roughly 25–30 tool calls unless exhaustive depth was requested; if the
cluster is too large for that, report on the flagged subset and name the
namespaces that went uninspected.

## Step 3 — Triage

Thresholds below are starting points, not laws. A number that is alarming in a
production namespace may be routine in a lab — weigh the cluster's apparent
purpose before escalating. Classify each finding as Critical, Warning, or Worth
noticing, and keep "this is broken" distinct from "this is unusual but might be
deliberate." A single-replica Deployment is a design choice in a lab and an
incident waiting to happen in production; say which you think it is and why.

### Availability

- Pods not Running or Succeeded: `CrashLoopBackOff`, `ImagePullBackOff`,
  `ErrImagePull`, `Pending`, `Error`, `Evicted`, `Init:*`, `CreateContainerError`
- `ContainerCreating` beyond a few minutes — usually a volume or image problem
  wearing a different mask
- `Terminating` past the grace period — a stuck finalizer or unresponsive kubelet
- Ready replicas below desired on any Deployment or StatefulSet
- DaemonSet not on every eligible node; check whether taints explain the gap
  before flagging
- Restart count above 5, or **any** restart in the last hour. Recent beats
  cumulative: a pod with 200 restarts that stabilized last week matters less
  than one that restarted twice in ten minutes
- Pods much younger than their neighbours in an otherwise stable namespace —
  something recycled and nobody noticed
- Rollouts stuck partway, or an old `progressDeadlineExceeded` condition

### Resources

- Nodes above ~80% CPU or memory allocation, or reporting `MemoryPressure`,
  `DiskPressure`, `PIDPressure`, or `NotReady`
- `OOMKilled` in any container's last state — always Critical; name the
  container and its memory limit
- Usage pinned at the limit: throttling for CPU, imminent OOM for memory
- Usage far below requests (under ~10% sustained) — waste, and it distorts
  scheduling for everything else on the node
- Containers with no requests or limits. Best-effort pods are evicted first
  under pressure, so this is a stability issue, not just hygiene
- Pods Pending purely on capacity; aggregate requests exceeding node allocatable

### Storage

- PVCs `Pending` or unbound — check the StorageClass exists and its provisioner
  is running
- `FailedMount` or `FailedAttachVolume` events, especially repeated
- Volumes near capacity where the data is available
- ReadWriteOnce volumes on multi-replica workloads — a rollout will deadlock
- No default StorageClass where workloads clearly expect one

### Configuration smells

- `:latest` or untagged images — unreproducible, and rollbacks do not work
- Images from unexpected registries relative to the rest of the cluster
- Single-replica Deployments serving something that looks like a critical path
- Missing liveness or readiness probes on long-running services; readiness is
  the one that breaks rollouts silently
- Multi-replica critical workloads with no PodDisruptionBudget — a node drain
  takes them all out at once
- No anti-affinity on replicas that all landed on one node
- Failed Jobs; CronJobs with no successful run in their expected window
- Suspended CronJobs nobody suspended on purpose
- Deprecated API versions still in use
- Orphaned resources with no owner reference, or resources stuck behind a
  finalizer
- Secrets or ConfigMaps referenced but absent
- ServiceAccounts with cluster-admin bindings on ordinary workloads
- Services with no matching endpoints — usually a selector typo
- Ingress with no backend, or TLS secrets expired or expiring soon

### Events

The cheapest signal in the cluster and the most commonly skipped. Weight by
recurrence: something firing 40 times matters more than something that fired once.

- `FailedScheduling` — read the reason; insufficient CPU, taints, and affinity
  failures have completely different fixes
- `Unhealthy` — probe failures, with the probe's own error text
- `BackOff`, `Failed`, `FailedCreate`, `FailedKillPod`, `NetworkNotReady`
- `NodeNotReady`, `Rebooted`, `NodeHasInsufficientMemory`
- Any warning class appearing across multiple namespaces at once — that is a
  cluster-level fault, not a workload-level one, and should be reported as such

### GPU signals — only if the cluster advertises GPUs

Generic checks miss GPU problems entirely: a starved queue looks like a few
ordinary Pending pods, and an unhealthy device plugin looks like a node that
quietly stopped receiving work.

- Compare `nvidia.com/gpu` allocatable against requests per node. GPUs are not
  overcommittable — 4 allocatable and 5 requested means one pod waits forever
- Pods Pending solely on GPU capacity; the `FailedScheduling` message names the
  resource, so distinguish this from CPU/memory scheduling failure
- Nodes advertising zero allocatable GPUs despite having hardware — almost
  always the device plugin, not the hardware
- GPU nodes missing their taint, so CPU-only workloads squat on them
- Device plugin and GPU Operator components (driver daemonset, container
  toolkit, DCGM exporter, NFD) not Running on every GPU node
- MIG configuration inconsistent across nodes in the same pool
- GPUs allocated but idle — the most expensive waste in the cluster. If DCGM
  metrics are reachable, compare utilization against allocation; if not, note
  that utilization could not be verified rather than assuming
- Long-running notebooks or dev shells holding GPUs — flag under "Worth
  noticing" rather than Critical; it may be exactly what someone intended
- Signatures worth surfacing verbatim: `CUDA error: no kernel image is
  available` (driver/toolkit mismatch), CUDA OOM (GPU memory exhaustion, which
  Kubernetes memory limits neither cover nor report), `nvidia-container-cli:
  initialization error` (runtime not configured on that node), Xid errors in
  node logs (hardware or driver fault)

## Step 4 — Report

Full report structure:

```markdown
# Cluster Health Report — <cluster> — <date>

## Summary
3–5 bullets. Overall state, and whether anything needs attention today.

## Critical
Broken or imminently breaking. Per finding:
namespace/kind/name · symptom · evidence · likely cause · suggested fix

## Warnings
Degraded or risky, still serving traffic.

## Worth noticing
Not failures: waste, drift, config smells, capacity trends, anything odd
for a cluster of this shape.

## Inventory
| Namespace | Workload | Kind | Ready | Restarts | CPU vs req | Mem vs req | Age |
Sorted worst-first.

## Recommended actions
Ordered by impact. Each with the exact command or manifest change.

## Coverage
What was inspected, what was skipped, and anything unreadable due to RBAC
or a missing metrics API.
```

In triage mode, drop everything except the finding, its evidence, and the fix.

## Evidence discipline

Every claim traces to something observed: an event message, an exit code, a log
line, a metric, a field in a spec. If it was not observed, mark it "not
verified" rather than inferring — a report that quietly guesses is worse than
one with gaps, because the user cannot tell which parts to trust.

Skip healthy, boring workloads in the narrative; they belong in the inventory
table and nowhere else. A report where everything is listed reads the same as a
report where nothing was checked.
