# Pin GPU node for Longhorn 

This is to allow NAI to only deploy file share under the GPU node.

⚠️ Note: can optionally disable scheduling for all other nodes in longhorn if other RWX also going to GPU node

1. Tag the GPU node and its disks

```bash
kubectl -n longhorn-system edit nodes.longhorn.io <gpu-node-name>
yaml
spec:
  tags:
    - gpu-node
  disks:
    disk1:
      tags:
        - gpu-node
    disk2:
      tags:
        - gpu-node
    # ...repeat for disk3, disk4
```
(Or via UI: Node → Edit Node and Disks → add tag gpu-node to the node and each disk.)

2. Create a StorageClass for RWX volumes that targets that tag
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-rwx-gpu
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  nodeSelector: "gpu-node"
  fsType: "ext4"
```
numberOfReplicas: "1" because if the GPU node is the only one tagged, that's the ceiling on how many replicas can be placed anyway — Longhorn requires distinct nodes per replica.

Any PVC created with accessModes: [ReadWriteMany] against this StorageClass will have its underlying volume's replica forced onto the tagged node.

3. Pin the share-manager pod itself (important — this is the actual NFS server for RWX)

Node/disk tags above control replica placement, but the share-manager pod is a regular Deployment scheduled by k8s, not automatically tied to the tag. To lock it (and instance-manager, backing-image-manager) to the GPU node:

```bash
kubectl label node <gpu-node-name> longhorn-system-managed=true
```
Then set the Longhorn global setting:

```bash
kubectl -n longhorn-system patch settings.longhorn.io system-managed-components-node-selector \
  --type merge -p '{"value":"longhorn-system-managed:true"}'
```
(Or via UI: Settings → General → "System Managed Components Node Selector".)

⚠️ Note: this setting is global — it affects all Longhorn system pods (instance-manager, share-manager, backing-image-manager, engine-image daemonset), not just RWX ones. Given you already want the whole node pinned as the sole Longhorn worker, this should be exactly what you want — but it means block/RWO volume instance-managers also get confined to this node too (which you'd already set up in Part 2 anyway).

4. Verify
```bash
kubectl -n longhorn-system get pods -o wide | grep share-manager
kubectl -n longhorn-system get replicas.longhorn.io -o wide
```

Confirm all share-manager pods and replicas show up only on the GPU node.