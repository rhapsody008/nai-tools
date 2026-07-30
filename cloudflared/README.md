# cloudflared

Deploys the Cloudflare Tunnel connector into the `cloudflare` namespace. Routing
(which public hostname maps to which internal service) is configured in the
Cloudflare Zero Trust dashboard under **Networks → Tunnels → Public Hostnames**,
not in this repo.

## Layout

- `namespace.yaml` — the `cloudflare` namespace
- `deployment.yaml` — the cloudflared connector, 2 fixed replicas (no HPA)
- `pdb.yaml` — keeps at least 1 replica up during node drains/upgrades
- `kustomization.yaml` — ties the above together for Flux/Kustomize

## The tunnel token is NOT in this repo

`deployment.yaml` expects a Secret named `tunnel-token` (key `TUNNEL_TOKEN`)
to already exist in the `cloudflare` namespace. Flux never creates it — you
do, once, directly against the cluster.

## Notes

- Image tag is pinned (`2026.7.3`) rather than `latest` — bump deliberately.
- `replicas: 2` with no autoscaler: HPA scale-down kills active tunnel
  connections, so this is intentionally fixed.
- No Kubernetes `Ingress`/`Service` object is needed for tunnel-routed
  services — adding one duplicates routing that the tunnel already owns.
