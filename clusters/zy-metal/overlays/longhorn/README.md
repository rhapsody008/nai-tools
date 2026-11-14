# Pin GPU node for Longhorn 

This is to allow NAI to only deploy file share under the GPU node.

⚠️ Note: can optionally disable scheduling for all other nodes in longhorn if other RWX also going to GPU node

1. Register and tag the GPU node's disks (before Longhorn is installed)

The disks must already be mounted persistently on the node (on zy-metal: systemd mount units `var-mnt-longhorn1..4.mount`, see [../../rollout/phase1-plan.md](../../rollout/phase1-plan.md) step B1).

`createDefaultDiskLabeledNodes: true` is set in [helmrelease-values.yaml](helmrelease-values.yaml), so Longhorn creates disks only on nodes carrying the label below, using the disks listed in the annotation. Set them once, by hand, **before** Longhorn is installed; Longhorn reads them only when it first registers the node. They are deliberately not Flux-managed (Flux owning a Node object risks prune deleting it).

```bash
kubectl annotate node <gpu-node-name> node.longhorn.io/default-disks-config='[{"name":"longhorn-1","path":"/var/mnt/longhorn1","allowScheduling":true,"storageReserved":53687091200,"tags":["gpu-node"]},{"name":"longhorn-2","path":"/var/mnt/longhorn2","allowScheduling":true,"storageReserved":0,"tags":["gpu-node"]},{"name":"longhorn-3","path":"/var/mnt/longhorn3","allowScheduling":true,"storageReserved":0,"tags":["gpu-node"]},{"name":"longhorn-4","path":"/var/mnt/longhorn4","allowScheduling":true,"storageReserved":0,"tags":["gpu-node"]}]'
kubectl annotate node <gpu-node-name> node.longhorn.io/default-node-tags='["gpu-node"]'
kubectl label node <gpu-node-name> node.longhorn.io/create-default-disk=config
```
`storageReserved` (bytes) is space Longhorn never schedules into: 50 GiB on disk 1, which also holds non-Longhorn data (`nai/`, `llmbench/`). Longhorn's own minimal-available guard (25% by default) protects the rest.

After Longhorn is installed, later changes go on the Longhorn node object instead (`kubectl -n longhorn-system edit nodes.longhorn.io <gpu-node-name>`, or the UI: Node → Edit Node and Disks).

2. Create a StorageClass for RWX volumes that targets that tag - see gpu-storageclass.yaml

numberOfReplicas: "1" because if the GPU node is the only one tagged, that's the ceiling on how many replicas can be placed anyway — Longhorn requires distinct nodes per replica.

Any PVC created with accessModes: [ReadWriteMany] against this StorageClass will have its underlying volume's replica forced onto the tagged node.

3. Pin the share-manager pod itself (important — this is the actual NFS server for RWX)

Node/disk tags above control replica placement, but the share-manager pod is a regular Deployment scheduled by k8s, not automatically tied to the tag. To lock it (and instance-manager, backing-image-manager) to the GPU node:

Set the label `node.cluster.x-k8s.io/longhorn-system-managed=true` on the GPU node pool in NKP. Cluster API only syncs Machine labels to the Node for allowed prefixes (`node.cluster.x-k8s.io/`, `node-role.kubernetes.io/`, and NKP's `nvidia.com/`, `nutanix.com/`, `amd.com/`); an unprefixed label like `longhorn-system-managed` stays on the Machine. For a node outside a node pool:

```bash
kubectl label node <gpu-node-name> node.cluster.x-k8s.io/longhorn-system-managed=true
```
The Longhorn global setting "System Managed Components Node Selector" is set in [helmrelease-values.yaml](helmrelease-values.yaml) (`defaultSettings.systemManagedComponentsNodeSelector: node.cluster.x-k8s.io/longhorn-system-managed:true`), so nothing to set by hand.

⚠️ Note: this setting is global — it affects all Longhorn system pods (instance-manager, share-manager, backing-image-manager, engine-image daemonset, CSI components), not just RWX ones. Given you already want the whole node pinned as the sole Longhorn worker, this should be exactly what you want — but it means block/RWO volume instance-managers also get confined to this node too (which you'd already set up in Part 2 anyway).

⚠️ Longhorn only applies a change to this setting while no Longhorn volumes are attached. Make sure the GPU node is labelled before Longhorn is installed (or before any PVCs exist); changing it later means detaching all volumes first. Check the live value with:

```bash
kubectl -n longhorn-system get settings.longhorn.io system-managed-components-node-selector
```

4. Verify
```bash
kubectl -n longhorn-system get pods -o wide | grep share-manager
kubectl -n longhorn-system get replicas.longhorn.io -o wide
```

Confirm all share-manager pods and replicas show up only on the GPU node.