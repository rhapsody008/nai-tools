# zy-metal phase 2 progress

Phase 1: [phase1-progress.md](phase1-progress.md). Scope: cloudflared, kubernetes-mcp-server, openwebui. Update this file after every step: status, date, and the key evidence line.

Status values: `todo`, `in progress`, `done`, `blocked`, `skipped`.

## Steps

| Step | What | Mark | Status | Date | Evidence / notes |
|---|---|---|---|---|---|
| 1 | cloudflared enabled ahead of phase 2 | CHANGE | done | 2026-11-15 | Commit `dae4dbf`. Tunnel `zy-metal-tunnel` (renamed in the dashboard, token rotated). Secret `cloudflared-credentials` (key `credentials.json`) applied manually. 2 replicas Running, tunnel registered (http2) |
| 2 | NAI installed (by ZY, outside Flux) | CHANGE | done | 2026-11-15 | ClickHouse was Pending (PV on `nutanix-volume` provisioned for the L40 node, which can't attach it; md-0 workers short on CPU). Resolved after md-0 grew to 6 workers. Model endpoint running in `nai-admin` on `longhorn-rwx-gpu` |
| 3 | Rename k8s-mcp hostname `zy-gpu-k8s-mcp` → `zy-metal-k8s-mcp` | CHANGE | done | 2026-11-15 | cloudflared `config.yaml`; Cloudflare Access app (Bypass, lab egress IP only) set up by ZY |
| 4 | kubernetes-mcp-server chart 0.1.3 | CHANGE | done | 2026-11-16 | Pushed `488c728`. HelmRelease "Helm install succeeded … kubernetes-mcp-server@0.1.3"; pod Running on `zy-metal-l40-node`; ClusterRoleBinding `kubernetes-mcp-server-kubernetes-mcp-server-admin` → cluster-admin |
| 5 | OpenWebUI image `v0.13.3`, requests 250m/1Gi | CHANGE | done | 2026-11-16 | No node affinity (accepted risk). Pod landed on md-0 `tszcx`; PVC `openwebui-data` Bound (`nutanix-volume`, 20Gi). First start ~4 min (DB migrations) |
| 6 | Verify Flux, pods, PVC, cloudflared roll | READ-ONLY | done | 2026-11-16 | Kustomizations zy-tools, cloudflared, kubernetes-mcp-server, openwebui Ready at `488c728`. cloudflared rolled to `cloudflared-config-hm9hdkb9fc` (old ConfigMap pruned), 4 connections registered, no errors |
| 7 | Verify URLs; k8s-mcp blocked from outside | READ-ONLY | done | 2026-11-16 | From ZY's Mac: openwebui `/health` 200 `{"status":true}`, `/` 200. zy-metal-k8s-mcp `/healthz` 403 Cloudflare Access page (Mac egress differs from the lab: Mac hits SIN edge, tunnel connects via MRS) — blocking works. Old `zy-gpu-k8s-mcp` hostname → 404 from cloudflared catch-all (no route). NAI connector to k8s-mcp added by ZY and working, so the Bypass matches the lab egress |
| 8 | OpenWebUI admin registered; manual settings and skills (apps/openwebui/README.md) | CHANGE | done | 2026-11-16 | Done by ZY; signup locked down |
| 9 | Final sanity check; remove public Longhorn UI route | CHANGE | done | 2026-11-16 | All nodes Ready, Flux/HelmReleases Ready at `488c728`, no failing pods, all PVCs Bound, Longhorn volume healthy, NAI ClickHouse Completed, qwen predictor on L40 with `longhorn-rwx-gpu` PVC (closes phase 1 D-8). Longhorn UI was public with no auth → route removed; Longhorn managed via its CRs from now on. Follow-ups: `nai-docker-regcred` missing in `nai-admin` (FailedToRetrieveImagePullSecret; running pod unaffected); delete stale `zy-gpu-k8s-mcp` DNS record |

## Decision log

| Date | Decision |
|---|---|
| 2026-11-15 | cloudflared: one tunnel per cluster; routes stay in git |
| 2026-11-15 | k8s-mcp exposed publicly (NAI needs a public-trusted HTTPS URL); protected by Cloudflare Access Bypass limited to the lab egress IP |
| 2026-11-15 | OpenWebUI: no node affinity; signup settings unchanged, ZY registers admin right after deploy |
| 2026-11-15 | Keep the jaeger route for later |
