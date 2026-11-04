# Tools Onboarding using Flux

Deployed to the workspace namespace (`zy-gpu-workspace`) of each workload cluster. Create the namespace if needed.

## Layout

```
apps/<tool>/                  # cluster-agnostic kustomize base per tool
clusters/<cluster>/
  tools-flux.yaml             # bootstrap, applied manually once per cluster
  flux/                       # Flux Kustomizations synced to this cluster
    kustomization.yaml        # list of tools enabled on this cluster
  overlays/<tool>/            # cluster-specific patches (only when needed)
manual/                       # not managed by Flux, applied by hand
src/                          # temporary shim for zy-gpu, see below
```

Clusters:
- `zy-gpu`: all tools, including cloudflared.
- `zy-gpu-metal`: all tools except cloudflared (stays on zy-gpu for now).

## Prerequisite

Cloudflare specific: create the secret before the deployment. Create the namespace if not exist.

```
kubectl create namespace cloudflared

kubectl create secret generic cloudflared-credentials \
  --namespace=cloudflared \
  --from-file=credentials.json=czxxz.json \
  --dry-run=client -o yaml | kubectl apply -f -
```

## Bootstrap

Bootstrap Flux using the cluster's tools-flux.yaml. Do once per cluster.

```
kubectl apply -f clusters/<cluster>/tools-flux.yaml
```

### zy-gpu migration note

zy-gpu was bootstrapped with the old root `tools-flux.yaml` (path `./src`). `src/kustomization.yaml` points at `clusters/zy-gpu/flux` so it keeps working unchanged. To finish the migration:
1. `kubectl apply -f clusters/zy-gpu/tools-flux.yaml` (updates the `zy-tools` root Kustomization path in place).
2. Then delete `src/` in a later commit.

## Adding new tools
1. Create `apps/<tool>/` with a kustomization.yaml and other k8s resources.
2. For each cluster that should run it, add `clusters/<cluster>/flux/<tool>.yaml` and list it in that cluster's `flux/kustomization.yaml`.
3. If a cluster needs different values, add `clusters/<cluster>/overlays/<tool>/` on top of the base and point the Flux Kustomization path there.

## NAI Pinning for GPU node - Kyverno + Longhorn

Longhorn is only used for the nai-admin RWX volumes below; everything else uses the default storageclass (nutanix-volume).

Longhorn StorageClass: set to deploy on labels gpu-node
System Managed Components Node Selector in Longhorn setting: set to longhorn-system-managed:true
GPU Node label: set to longhorn-system-managed:true

Kyverno policy: pods in namepace nai-admin will be scheduled on nodes with label longhorn-system-managed:true

Kyverno is split into `apps/kyverno/controller` and `apps/kyverno/policies`, because the ClusterPolicy CRD comes from the chart. New clusters sync them as two Flux Kustomizations (`kyverno-policies` dependsOn `kyverno`); zy-gpu syncs both together via `apps/kyverno`.
