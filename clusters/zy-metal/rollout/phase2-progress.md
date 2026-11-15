# zy-metal phase 2 progress

Phase 1: [phase1-progress.md](phase1-progress.md). Scope: cloudflared, kubernetes-mcp-server, openwebui. Update this file after every step: status, date, and the key evidence line.

Status values: `todo`, `in progress`, `done`, `blocked`, `skipped`.

## Steps

| Step | What | Mark | Status | Date | Evidence / notes |
|---|---|---|---|---|---|
| 1 | cloudflared enabled ahead of phase 2 | CHANGE | done | 2026-11-15 | Commit `dae4dbf`. Tunnel `zy-metal-tunnel` (renamed in the dashboard, token rotated). Secret `cloudflared-credentials` (key `credentials.json`) applied manually. 2 replicas Running, tunnel registered (http2) |
| 2 | NAI installed (by ZY, outside Flux) | CHANGE | done | 2026-11-15 | ClickHouse was Pending (PV on `nutanix-volume` provisioned for the L40 node, which can't attach it; md-0 workers short on CPU). Resolved after md-0 grew to 6 workers. Model endpoint running in `nai-admin` on `longhorn-rwx-gpu` |
| 3 | Rename k8s-mcp hostname `zy-gpu-k8s-mcp` → `zy-metal-k8s-mcp` | CHANGE | done | 2026-11-15 | cloudflared `config.yaml`; Cloudflare Access app (Bypass, lab egress IP only) set up by ZY |
| 4 | kubernetes-mcp-server chart 0.1.3 | CHANGE | in progress | 2026-11-15 | cluster-admin kept; public only through the Access-protected hostname |
| 5 | OpenWebUI image `v0.13.3`, requests 250m/1Gi | CHANGE | in progress | 2026-11-15 | No node affinity (accepted risk: `nutanix-volume` PVC provisioned for the L40 node, as with ClickHouse) |
| 6 | Verify Flux, pods, PVC, cloudflared roll | READ-ONLY | todo | | |
| 7 | Verify URLs in-lab; k8s-mcp blocked from outside | READ-ONLY | todo | | |
| 8 | OpenWebUI admin registered; manual settings and skills (apps/openwebui/README.md) | CHANGE | todo | | |

## Decision log

| Date | Decision |
|---|---|
| 2026-11-15 | cloudflared: one tunnel per cluster; routes stay in git |
| 2026-11-15 | k8s-mcp exposed publicly (NAI needs a public-trusted HTTPS URL); protected by Cloudflare Access Bypass limited to the lab egress IP |
| 2026-11-15 | OpenWebUI: no node affinity; signup settings unchanged, ZY registers admin right after deploy |
| 2026-11-15 | Keep the jaeger route for later |
