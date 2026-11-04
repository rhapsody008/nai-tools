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

## Onboarding a new cluster

1. **Prepare the cluster.** Attach it to the NKP workspace so Flux (kommander-flux) is running in `zy-gpu-workspace`.
2. **Create the cluster folder** by copying an existing one, e.g. `cp -R clusters/zy-gpu-metal clusters/<cluster>`.
   - `tools-flux.yaml`: set the root path to `./clusters/<cluster>/flux`.
   - `flux/kustomization.yaml`: keep only the tools you want to start with. Add the rest one at a time later.
   - `flux/*.yaml`: on a fresh cluster use `apps/kyverno/controller` + `apps/kyverno/policies` (as in zy-gpu-metal), not `apps/kyverno`.
   - `overlays/`: patch anything cluster-specific (e.g. openwebui `CLUSTER_NAME`). Point the matching `flux/<tool>.yaml` path at the overlay.
3. **Check the render** before committing:
   ```
   kubectl kustomize clusters/<cluster>/flux
   kubectl kustomize clusters/<cluster>/overlays/<tool>
   ```
4. **Create secrets** needed by the enabled tools (see Prerequisite), then commit and push.
5. **Bootstrap:** `kubectl apply -f clusters/<cluster>/tools-flux.yaml`
6. **Verify:** `kubectl -n zy-gpu-workspace get kustomizations,gitrepositories` should show `zy-tools` and each tool Ready.
7. **Longhorn GPU pinning** (if longhorn + kyverno are enabled): tag the GPU node and its disks `gpu-node` and set the labels described in [apps/longhorn/README.md](apps/longhorn/README.md) and below, before NAI creates PVCs in `nai-admin`.
8. **Roll more tools over** by adding lines to `flux/kustomization.yaml` and pushing.

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

Kyverno is split into `apps/kyverno/controller` and `apps/kyverno/policies`, because the ClusterPolicy CRD comes from the chart. New clusters sync them as two Flux Kustomizations (`kyverno-policies` dependsOn `kyverno`); zy-gpu syncs both together via `apps/kyverno` since kyverno was already installed there.
