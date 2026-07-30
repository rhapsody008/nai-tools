# cloudflared

Runs a **locally-managed** Cloudflare Tunnel — ingress routing lives in
`config.yaml` in this repo (versioned, reviewable in PRs) instead of the
Cloudflare dashboard.

## Layout

- `config.yaml` — tunnel ID + ingress rules (hostname → service). Plain data,
  turned into a ConfigMap by the `configMapGenerator` in `kustomization.yaml`.
  Kustomize hashes the content into the ConfigMap name, so any edit here
  forces a pod rollout automatically — no manual restart needed.
- `deployment.yaml` — the cloudflared connector, 2 fixed replicas (no HPA),
  mounts `config.yaml` and the `cloudflared-credentials` Secret
- `pdb.yaml` — keeps at least 1 replica up during node drains/upgrades
- `kustomization.yaml` — ties the above together for Flux/Kustomize

## Editing routes

Edit the `ingress` list in `config.yaml` and push. Flux picks it up on the
next reconcile (or force it: `flux reconcile kustomization cloudflared`).
Keep the trailing `- service: http_status:404` rule last — cloudflared
requires a catch-all as the final rule.

## Credentials are NOT in this repo

`deployment.yaml` expects a Secret named `cloudflared-credentials` (key
`credentials.json`) in the `cloudflare` namespace. Flux never creates it —
you do, once, directly in NKP console.

It'll ask for a path to an existing `credentials.json` (dropped by
`cloudflared tunnel create` into `~/.cloudflared/`), or you can paste the
AccountTag / TunnelID / TunnelSecret values individually. Re-run it any time
you rotate credentials.

Also fill in the real Tunnel ID in `config.yaml`'s `tunnel:` field — that one
isn't sensitive on its own, so it's fine to commit.

## Notes

- Image tag is pinned (`2026.7.3`) rather than `latest` — bump deliberately.
- `replicas: 2` with no autoscaler: HPA scale-down kills active tunnel
  connections, so this is intentionally fixed.