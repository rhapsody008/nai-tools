# Tools Onboarding using Flux

Deployed to the workspace namespace (`zy-gpu-workspace`) of each workload cluster. Create the namespace if needed.

## Layout

```
apps/<tool>/                  # base install only: namespace, chart source, chart/manifests, no custom values
clusters/<cluster>/
  tools-flux.yaml             # bootstrap, applied manually once per cluster
  flux/                       # Flux Kustomizations synced to this cluster, one per tool
    kustomization.yaml        # list of tools enabled on this cluster
  overlays/<tool>/            # everything cluster-specific: Helm values, configs, policies, extra resources
manual/                       # not managed by Flux, applied by hand
```

Every Flux Kustomization points at `clusters/<cluster>/overlays/<tool>`, never straight at `apps/`. The overlay pulls in the base and adds the customizations. Helm values go in `helmrelease-values.yaml`, a patch on the base HelmRelease.

Clusters:
- `zy-metal`: see `clusters/zy-metal/flux/kustomization.yaml` for the tools enabled.

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
2. **Create the cluster folder** by copying an existing one, e.g. `cp -R clusters/zy-metal clusters/<cluster>`.
   - `tools-flux.yaml`: set the root path to `./clusters/<cluster>/flux`, and the workspace namespace (the two `metadata.namespace` fields and `targetNamespace`). It's the only place the workspace namespace is set; `flux/*.yaml` inherit it via `targetNamespace`.
   - `flux/kustomization.yaml`: keep only the tools you want to start with. Add the rest one at a time later.
   - `flux/*.yaml`: set each path to `./clusters/<cluster>/overlays/<tool>`.
   - `overlays/`: adjust the values and configs for the new cluster (e.g. openwebui `CLUSTER_NAME` in `cluster-config.yaml`, cloudflared routes in `config.yaml`).
3. **Check the render** before committing:
   ```
   kubectl kustomize clusters/<cluster>/flux
   kubectl kustomize clusters/<cluster>/overlays/<tool>
   ```
4. **Create secrets** needed by the enabled tools (see Prerequisite), then commit and push.
5. **Bootstrap:** `kubectl apply -f clusters/<cluster>/tools-flux.yaml`
6. **Verify:** `kubectl -n zy-gpu-workspace get kustomizations,gitrepositories` should show `zy-tools` and each tool Ready.
   - **OpenWebUI skills are not deployed by Flux.** They live in OpenWebUI's database, so a new OpenWebUI starts with none. Add them by hand from [apps/openwebui/skills/](apps/openwebui/skills/) (see [apps/openwebui/README.md](apps/openwebui/README.md)).
7. **Longhorn GPU pinning** (if longhorn + kyverno are enabled): **before enabling Longhorn**, mount the GPU node's disks persistently and set the node label and annotations that register and tag them `gpu-node`, as described in [clusters/zy-metal/overlays/longhorn/README.md](clusters/zy-metal/overlays/longhorn/README.md) and below.
8. **Roll more tools over** by adding lines to `flux/kustomization.yaml` and pushing.

## Adding new tools
1. Create `apps/<tool>/` with the base install only (namespace, chart source, HelmRelease without custom values, or plain manifests).
2. For each cluster that should run it, add `clusters/<cluster>/overlays/<tool>/` with a kustomization.yaml that references `../../../../apps/<tool>` plus the cluster's values/configs.
3. Add `clusters/<cluster>/flux/<tool>.yaml` pointing at that overlay and list it in the cluster's `flux/kustomization.yaml`.

## NAI Pinning for GPU node - Kyverno + Longhorn

Longhorn is only used for the nai-admin RWX volumes below; everything else uses the default storageclass (nutanix-volume).

Longhorn StorageClass: set to deploy on labels gpu-node
System Managed Components Node Selector in Longhorn setting: node.cluster.x-k8s.io/longhorn-system-managed:true, set via `defaultSettings` in [clusters/zy-metal/overlays/longhorn/helmrelease-values.yaml](clusters/zy-metal/overlays/longhorn/helmrelease-values.yaml). This pins all Longhorn system components, including share-manager (the RWX NFS server), to the GPU node. Only applied while no Longhorn volumes are attached, so label the node before installing Longhorn.
GPU Node label: set to node.cluster.x-k8s.io/longhorn-system-managed:true (set on the NKP node pool; unprefixed labels are not synced to nodes by Cluster API)

Kyverno policy: pods in namepace nai-admin will be scheduled on nodes with label node.cluster.x-k8s.io/longhorn-system-managed:true, and nai-admin PVCs without a storageClassName default to longhorn-rwx-gpu.

Kyverno policies live in `clusters/<cluster>/overlays/kyverno-policies/` and are synced as a separate Flux Kustomization (`kyverno-policies` dependsOn `kyverno`), because the ClusterPolicy CRD comes from the kyverno chart.
