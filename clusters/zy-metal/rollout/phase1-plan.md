# zy-metal phase 1 rollout plan: gpu-operator, Longhorn, Kyverno

Status tracking lives in [phase1-progress.md](phase1-progress.md). This file is the plan; change it only when the plan changes.

- **Cluster:** zy-metal (NKP 2.19.1, k8s v1.36.2, workspace `zy-gpu-workspace`)
- **Scope:** gpu-operator, longhorn, kyverno, kyverno-policies. Phase 2 (cloudflared, kubernetes-mcp, openwebui) is reviewed separately after phase 1 is verified.
- **Way of working:** one step at a time, verify before the next. Every CHANGE or DESTRUCTIVE step needs ZY's approval. kubectl runs from `ops/` as `kubectl <verb> ... --kubeconfig kubeconfigs/zy-metal.conf`.
- **Marks:** READ-ONLY (may run), CHANGE (approval), DESTRUCTIVE (approval, every time).

## Goal

- All four tools reconciled by kommander-flux from pushed `main`.
- `zy-metal-l40-node` is the only Longhorn storage node, with its 4 SSD disks tagged `gpu-node`.
- The node advertises `nvidia.com/gpu: 4`.
- ClusterPolicy `pin-nai-admin-to-l40` Ready.

## Key facts this plan relies on

- **GPU node** `zy-metal-l40-node` (172.16.0.90, host-agent bare metal): 4x L40, 96 CPU, ~504Gi.
  - Kairos immutable image (`kubernetes-os-ubuntu-24.04-nkp-v2.19.1`), kernel 6.8.0-134. `/etc` and `/var` are tmpfs overlays except listed persistent paths (`/etc/systemd`, `/etc/containerd`, `/var/lib/longhorn`, `/opt`, ...). `/etc/fstab` and `/etc/multipath*` are **not** persistent; `/oem` cloud-configs are.
  - containerd 2.3.4 (CAPX nodes run 2.3.1-d2iq.1), config v3, `imports = []`, CDI enabled. `certs.d/_default/hosts.toml` sends all registries to `registry.ntnxlab.local/library` first; upstream fallback is expected but unverified.
  - No kernel headers, no compiler on the host. Secure Boot off. nouveau not loaded. Direct internet works from the node.
  - Disks: `nvme0n1` = OS. `sda`..`sdd` = 745G SSDs, each one xfs partition labelled `longhorn-1..4`, already initialised by an earlier Longhorn (`longhorn-disk.cfg`, empty `replicas/`). `sda1` also holds `nai/` and `llmbench/` (~2.2 GiB). **Keep all data, no reformat.**
  - multipathd active, does not claim `sda..sdd` (no WWID). Longhorn's own iSCSI devices need a blacklist.
  - Label `node.cluster.x-k8s.io/longhorn-system-managed=true` already set.
- **Cluster:** NFD already installed (ns `node-feature-discovery`). Only StorageClass `nutanix-volume` (default). `zy-gpu-workspace` is not a Project namespace, so the Flux service-account Gatekeeper constraints don't apply.

## Decisions

| # | Decision | Choice |
|---|---|---|
| D-1 | gpu-operator driver | `driver.enabled=true` (driver container builds the module on the node) |
| D-2 | Disks | Keep existing xfs partitions and data; no wipe |
| D-3 | Longhorn disk registration | `createDefaultDiskLabeledNodes: true` in git + one-time node label/annotations (not Flux-managed: Flux owning a Node object risks prune deleting it) |
| D-4 | Rollout | One tool per push: gpu-operator → longhorn → kyverno + kyverno-policies |
| D-5 | Longhorn replica counts | CSI sidecar replica counts and `defaultClassReplicaCount` = 1 (single storage node) |
| D-6 | Kyverno failure mode | `failurePolicy: Ignore` on `pin-nai-admin-to-l40` |
| D-7 | Longhorn reserved space per disk | 50 GiB (`53687091200`) on disk 1 (holds `nai/`, `llmbench/`), 0 on disks 2–4 (Longhorn's 25% minimal-available guard covers them) |
| D-8 | `nai-admin` policy functional test | Defer to the NAI install |
| D-9 | Second GPU-node reboot after gpu-operator | Yes |
| D-10 | Push of nai-tools | Only on ZY's explicit go |

## P. Pre-checks (READ-ONLY)

Stop and report if any fails.

- **P1 repo and charts** (workstation)
  - `git ls-remote https://github.com/rhapsody008/nai-tools.git main` without credentials → returns a SHA (repo public; Flux has no secretRef).
  - `helm show chart gpu-operator --repo https://helm.ngc.nvidia.com/nvidia --version v26.7.2`
  - `helm show chart longhorn --repo https://charts.longhorn.io --version 1.12.1`
  - `helm show chart kyverno --repo https://kyverno.github.io/kyverno/` → pick the version to pin; confirm `kubeVersion` allows 1.36.
- **P2 cluster**
  - Flux CRD versions: HelmRelease `v2`, others `v1`.
  - GPU node NFD labels: `system-os_release.ID=ubuntu`, `VERSION_ID=24.04`, `kernel-version.full=6.8.0-134-generic` (driver image tag is built from these).
  - Namespace PSA labels on `node-feature-discovery` / `kube-system` → do new namespaces need `pod-security.kubernetes.io/enforce=privileged`?
  - `kubectl apply --dry-run=server -f clusters/zy-metal/tools-flux.yaml` → no Gatekeeper denial.
- **P3 GPU node** (`ssh nutanix@172.16.0.90`)
  - `/oem` and `/usr/local` writable and persistent? (toolkit installs to `/usr/local/nvidia`; else `toolkit.installDir: /opt/nvidia`).
  - Longhorn host tools: `iscsiadm`, `mount.nfs`/`mount.nfs4`, `cryptsetup`, iscsid active/enabled, modules `iscsi_tcp nfs nfsv4 dm_crypt`.
  - Ubuntu archive has `linux-headers-6.8.0-134-generic` (noble-updates / noble-security Packages index).
  - `lsblk` of sda..sdd: distinct labels and UUIDs, unmounted.
  - `systemctl cat kubelet.service` (unit name for `Before=`).
  - `/oem/22_ntp.yaml` (non-secret) to confirm the yip stage format. Do not read `24_user`, `42_nkp_userdata`, `99_nkp_bootstrap`.

## A. Git changes (nai-tools, local until A8)

- **A1 READ-ONLY** `helm show values longhorn --repo https://charts.longhorn.io --version 1.12.1 | grep -nE 'guaranteed|createDefaultDiskLabeledNodes|systemManagedComponentsNodeSelector|taintToleration|defaultDataPath'` → confirm key names.
- **A2 CHANGE** `apps/kyverno/helmrelease.yaml`: replace `version: "*"` with the P1 version.
- **A3 CHANGE** `overlays/gpu-operator/helmrelease-values.yaml`: add `nfd.enabled: false`. Add `toolkit.installDir: /opt/nvidia` only if P3 says `/usr/local` is unusable.
- **A4 CHANGE** `overlays/longhorn/helmrelease-values.yaml`:
  - delete `guaranteedEngineManagerCPU` and `guaranteedReplicaManagerCPU` (1.12 uses `guaranteedInstanceManagerCPU`, default 12% — the intended value);
  - add `createDefaultDiskLabeledNodes: true`;
  - set `csi.attacherReplicaCount`, `provisionerReplicaCount`, `resizerReplicaCount`, `snapshotterReplicaCount` and `persistence.defaultClassReplicaCount` to 1 (D-5).
- **A5 CHANGE** `overlays/kyverno-policies/pin-nai-cluster-policy.yaml`: failure policy Ignore (D-6). Use the field the pinned Kyverno version expects (`spec.webhookConfiguration.failurePolicy` on newer versions, `spec.failurePolicy` on older ones; check the CRD/docs for the A2 version).
- **A6 CHANGE** docs: Longhorn overlay README step 1 → label/annotation method (before install, not Flux-managed); `gpu-storageclass.yaml` comment (tags set before install); `README.md` onboarding step 7.
- **A7 CHANGE** `flux/kustomization.yaml`: comment out `longhorn.yaml`, `kyverno.yaml`, `kyverno-policies.yaml` (gpu-operator only for the first push).
- **A8** READ-ONLY render check (`kubectl kustomize clusters/zy-metal/flux` and each overlay); CHANGE commit; CHANGE **push only on ZY's go, after B and C**.
- Rollback: before push `git reset --soft HEAD~1`; after push `git revert` + push.

## B. GPU node host prep (`ssh nutanix@172.16.0.90`, all CHANGE unless marked)

- **B1 Mount units** `/etc/systemd/system/var-mnt-longhorn{1..4}.mount`:
  ```ini
  [Unit]
  Description=Longhorn data disk N (xfs label longhorn-N)
  Before=kubelet.service

  [Mount]
  What=/dev/disk/by-label/longhorn-N
  Where=/var/mnt/longhornN
  Type=xfs
  Options=defaults

  [Install]
  WantedBy=local-fs.target
  ```
  `sudo systemctl daemon-reload && sudo systemctl enable --now var-mnt-longhorn{1,2,3,4}.mount`.
  Verify: `findmnt -t xfs | grep longhorn` (4 mounts), `systemctl is-enabled`, `df -h /var/mnt/longhorn*`.
  Rollback: `disable --now`, remove units, `daemon-reload` (data untouched).
- **B2 READ-ONLY** `sudo cat /var/mnt/longhornN/longhorn-disk.cfg` for N=1..4 → 4 distinct `diskUUID`s, empty `replicas/`. Duplicates → stop, ask ZY. If the cfg has a `diskName`, reuse it in C1.
- **B3 Multipath blacklist** for Longhorn iSCSI devices, persistent via `/oem/95_longhorn.yaml` (initramfs stage writes `/etc/multipath/conf.d/longhorn.conf`; boot stage runs `multipathd reconfigure`), plus the same file applied live:
  ```
  blacklist {
      device {
          vendor "IET"
          product "VIRTUAL-DISK"
      }
  }
  ```
  Verify: `sudo multipath -t | grep -A3 IET`, multipathd active, `multipath -ll` unchanged.
  Rollback: remove both files, `multipathd reconfigure`.
- **B4** `sudo cp -a /etc/containerd/config.toml /etc/containerd/config.toml.pre-nvidia`.
- **B5 Reboot** (`sudo systemctl reboot`). Verify: node Ready (host-agent Machine may lag a few minutes — normal), 4 mounts back, `longhorn.conf` present, `systemctl --failed` empty, `systemd-analyze critical-chain kubelet.service` shows mounts before kubelet, pods on the node Running, `diff config.toml config.toml.pre-nvidia` (difference = bootstrap rewrites it on boot). If `longhorn.conf` is missing, move B3 to the `boot` stage and reboot again.

## C. Node label and annotations (CHANGE, before Longhorn is enabled)

```sh
kubectl annotate node zy-metal-l40-node node.longhorn.io/default-disks-config='[
  {"name":"disk1","path":"/var/mnt/longhorn1","allowScheduling":true,"storageReserved":<D-7>,"tags":["gpu-node"]},
  {"name":"disk2","path":"/var/mnt/longhorn2","allowScheduling":true,"storageReserved":<D-7>,"tags":["gpu-node"]},
  {"name":"disk3","path":"/var/mnt/longhorn3","allowScheduling":true,"storageReserved":<D-7>,"tags":["gpu-node"]},
  {"name":"disk4","path":"/var/mnt/longhorn4","allowScheduling":true,"storageReserved":<D-7>,"tags":["gpu-node"]}]' --kubeconfig kubeconfigs/zy-metal.conf
kubectl annotate node zy-metal-l40-node node.longhorn.io/default-node-tags='["gpu-node"]' --kubeconfig kubeconfigs/zy-metal.conf
kubectl label node zy-metal-l40-node node.longhorn.io/create-default-disk=config --kubeconfig kubeconfigs/zy-metal.conf
```
(Send the JSON on one line.) Verify annotations/labels, re-check after a few minutes that host-agent/CAPI didn't strip them. Rollback: same commands with `<key>-`. Longhorn reads these only on first node registration; later changes go on `nodes.longhorn.io`.

## D. Flux bootstrap and rollout (all CHANGE)

- **D1** Push (A8), then `kubectl apply -f <nai-tools>/clusters/zy-metal/tools-flux.yaml --kubeconfig kubeconfigs/zy-metal.conf`. Verify GitRepository `zy-tools` Ready at the pushed SHA (proves Flux egress to GitHub), Kustomizations `zy-tools` and `gpu-operator` Ready. Rollback (DESTRUCTIVE): `kubectl delete -f` (prune removes all children).
- **D2** Watch gpu-operator → run E1. Nothing further until E1 passes. Tool rollback (DESTRUCTIVE): comment out and push; restore `config.toml.pre-nvidia` + restart containerd if needed.
- **D3** Second reboot of the GPU node (D-9), re-run E1: driver rebuilds, toolkit config survives, 4 GPUs return (expect 5–10 min for the driver build).
- **D4** Uncomment `longhorn.yaml`, render, commit, push → E2. Rollback is costly: set Longhorn `deleting-confirmation-flag=true` first, then comment out and push (DESTRUCTIVE); disk data stays.
- **D5** Uncomment `kyverno.yaml` and `kyverno-policies.yaml` together, push → E3. Rollback: policies first, then kyverno. Emergency if the webhook blocks pods: delete Kyverno webhook configurations (DESTRUCTIVE).

## E. Verification (READ-ONLY unless marked)

- **E1 gpu-operator**
  - HelmRelease Ready (allow ~20 min); operator on a CAPX worker (proves CAPX image pulls); driver, toolkit, device-plugin, GFD, DCGM, validator on the L40 node; cuda-validator Completed; no NFD pods in `gpu-operator`.
  - Pull events: `nvcr.io/...` "Successfully pulled". ImagePullBackOff with mirror 404/auth → fallback doesn't work; stop.
  - Driver log (`ds/nvidia-driver-daemonset -c nvidia-driver-ctr`): headers for 6.8.0-134 installed, module built and loaded.
  - `nvidia-smi` in the driver pod: 4 L40, driver 615.71.09.
  - Node allocatable `nvidia.com/gpu` = 4, `nvidia.com/*` labels present.
  - Node: containerd active, `imports` set, `/etc/containerd/conf.d/` populated, `crictl info` shows the nvidia runtime; other pods on the node recovered after the containerd restart.
  - `kubectl get clusterpolicies.nvidia.com` → ready.
- **E2 Longhorn**
  - HelmRelease Ready; longhorn-manager on all nodes; instance-manager, engine-image, csi-plugin, CSI deployments only on the L40 node.
  - Settings: `system-managed-components-node-selector`, `create-default-disk-labeled-nodes`, `guaranteed-instance-manager-cpu`.
  - `nodes.longhorn.io zy-metal-l40-node`: tag `gpu-node`; 4 disks at `/var/mnt/longhorn1..4`, tagged, Ready/Schedulable, `diskUUID`s equal B2 (adopted, not re-created); no disks on other nodes; required-package/module/NFS conditions True.
  - StorageClasses: `longhorn-rwx-gpu` and `longhorn` present, `nutanix-volume` still the only default.
  - **RWX test (CHANGE):** `ops/work/lh-rwx-test.yaml` — ns `lh-rwx-test`, 1Gi RWX PVC on `longhorn-rwx-gpu`, busybox pod pinned to `zy-metal-l40-node` writing/reading `/data/probe`. Check Bound/Running, log `ok`, share-manager on the L40 node, one replica, new dir under `/var/mnt/longhorn*/replicas/`, `multipath -ll` shows no IET device. Cleanup (DESTRUCTIVE): delete the file's objects, confirm volume and replica gone.
- **E3 Kyverno**
  - Kustomizations `kyverno`, `kyverno-policies` Ready; HelmRelease at the pinned version; admission/background/cleanup/reports controllers Running.
  - `clusterpolicies.kyverno.io pin-nai-admin-to-l40` READY, no rule/autogen errors, failure policy Ignore.
  - Regression: server dry-run pod in `default` succeeds unchanged.
  - Functional test in `nai-admin`: deferred to the NAI install (D-8).

## Risks

1. **Internet from CAPX nodes and Flux is untested.** Only the GPU node was checked. D1/E1 reveal it; if they can't pull, images must go into `registry.ntnxlab.local/library`.
2. **Driver build** needs the Ubuntu archive and repeats on every boot (5–10 min before GPUs are allocatable). Fallbacks if headers can't be fetched (ZY decides): precompiled driver (likely a different version), private apt mirror via `driver.repoConfig`, or driver baked into the node image with `driver.enabled=false`.
3. **Toolkit vs immutable OS:** `/usr/local` persistence (P3) and whether boot rewrites `config.toml` (B5). The toolkit re-applies on start, at the cost of a containerd restart per boot.
4. **Longhorn disk adoption:** `diskName` matching in 1.12 unverified (B2/E2).
5. **Supportability:** upstream gpu-operator v26.7.2; do not also enable NKP's `nvidia-gpu-operator` platform app.
6. **Longhorn uninstall** is not a clean rollback (D4).
7. **Public repo:** if P1 confirms the repo is public, everything committed here (including this plan's IPs and hostnames) is public on push.
