# zy-metal tools rollout record (phases 1 and 2)

- **Record date:** 2026-11-16
- **Rollout window:** 2026-11-14 to 2026-11-16
- **Cluster:** zy-metal (NKP 2.19.1, workspace `zy-gpu-workspace`)
- **Status:** complete. Last commit of the rollout: `7ca27e2`.

This is the combined record of the zy-metal tools rollout, kept for reference. It replaces the separate `phase1-plan.md`, `phase1-progress.md` and `phase2-progress.md`; their content is unchanged below except heading levels and cross-links.

## Summary

| Phase | Dates | Scope | Outcome |
|---|---|---|---|
| 1 | 2026-11-14 to 2026-11-15 | gpu-operator, Longhorn, Kyverno + kyverno-policies | Complete at `e20cbc2`. Node advertises `nvidia.com/gpu=4`; Longhorn adopted the 4 existing L40 disks; nai-admin MutatingPolicies Ready |
| — | 2026-11-15 | NAI installed by ZY (outside Flux) | ClickHouse Pending fixed by growing md-0 to 6 workers; model endpoint on the L40 with `longhorn-rwx-gpu` |
| 2 | 2026-11-15 to 2026-11-16 | cloudflared, kubernetes-mcp-server, openwebui | Complete at `7ca27e2`. Tunnel `zy-metal-tunnel`; k8s-mcp behind Cloudflare Access (Bypass, lab egress IP); public Longhorn UI route removed |

Commits, in order: `f178da2` (host prep), `5f01f4d` (Longhorn), `8bc976f`, `e20cbc2` (Kyverno and policies), `d3c7c52` (phase 1 complete), `dae4dbf` (cloudflared), `488c728` (k8s-mcp, openwebui), `7ca27e2` (Longhorn route removed, phase 2 complete).

## Known follow-ups (not blocking)

- `nai-docker-regcred` missing in `nai-admin` (FailedToRetrieveImagePullSecret; the running model pod is unaffected).
- Stale Cloudflare DNS records `longhorn.yolocal.cloud` and `zy-gpu-k8s-mcp.yolocal.cloud` (return 404).
- `nutanix-volume` PVCs can be provisioned for the bare-metal L40 node, which can't attach them. OpenWebUI hit it once and rescheduled on its own; ClickHouse stayed stuck because the workers were also short on CPU. To be raised with PM if it recurs.
- `jaeger.yolocal.cloud` route kept for a later Jaeger deploy (502 until then).

---

## Phase 1 plan: gpu-operator, Longhorn, Kyverno

- **Cluster:** zy-metal (NKP 2.19.1, k8s v1.36.2, workspace `zy-gpu-workspace`)
- **Scope:** gpu-operator, longhorn, kyverno, kyverno-policies. Phase 2 (cloudflared, kubernetes-mcp, openwebui) is reviewed separately after phase 1 is verified.
- **Way of working:** one step at a time, verify before the next. Every CHANGE or DESTRUCTIVE step needs ZY's approval. kubectl runs from `ops/` as `kubectl <verb> ... --kubeconfig kubeconfigs/zy-metal.conf`.
- **Marks:** READ-ONLY (may run), CHANGE (approval), DESTRUCTIVE (approval, every time).

### Goal

- All four tools reconciled by kommander-flux from pushed `main`.
- `zy-metal-l40-node` is the only Longhorn storage node, with its 4 SSD disks tagged `gpu-node`.
- The node advertises `nvidia.com/gpu: 4`.
- ClusterPolicy `pin-nai-admin-to-l40` Ready.

### Key facts this plan relies on

- **GPU node** `zy-metal-l40-node` (172.16.0.90, host-agent bare metal): 4x L40, 96 CPU, ~504Gi.
  - Kairos immutable image (`kubernetes-os-ubuntu-24.04-nkp-v2.19.1`), kernel 6.8.0-134. `/etc` and `/var` are tmpfs overlays except listed persistent paths (`/etc/systemd`, `/etc/containerd`, `/var/lib/longhorn`, `/opt`, ...). `/etc/fstab` and `/etc/multipath*` are **not** persistent; `/oem` cloud-configs are.
  - containerd 2.3.4 (CAPX nodes run 2.3.1-d2iq.1), config v3, `imports = []`, CDI enabled. `certs.d/_default/hosts.toml` sends all registries to `registry.ntnxlab.local/library` first; upstream fallback is expected but unverified.
  - No kernel headers, no compiler on the host. Secure Boot off. nouveau not loaded. Direct internet works from the node.
  - Disks: `nvme0n1` = OS. `sda`..`sdd` = 745G SSDs, each one xfs partition labelled `longhorn-1..4`, already initialised by an earlier Longhorn (`longhorn-disk.cfg`, empty `replicas/`). `sda1` also holds `nai/` and `llmbench/` (~2.2 GiB). **Keep all data, no reformat.**
  - multipathd active, does not claim `sda..sdd` (no WWID). Longhorn's own iSCSI devices need a blacklist.
  - Label `node.cluster.x-k8s.io/longhorn-system-managed=true` already set.
- **Cluster:** NFD already installed (ns `node-feature-discovery`). Only StorageClass `nutanix-volume` (default). `zy-gpu-workspace` is not a Project namespace, so the Flux service-account Gatekeeper constraints don't apply.

### Decisions

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

### P. Pre-checks (READ-ONLY)

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

### A. Git changes (nai-tools, local until A8)

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

### B. GPU node host prep (`ssh nutanix@172.16.0.90`, all CHANGE unless marked)

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

### C. Node label and annotations (CHANGE, before Longhorn is enabled)

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

### D. Flux bootstrap and rollout (all CHANGE)

- **D1** Push (A8), then `kubectl apply -f <nai-tools>/clusters/zy-metal/tools-flux.yaml --kubeconfig kubeconfigs/zy-metal.conf`. Verify GitRepository `zy-tools` Ready at the pushed SHA (proves Flux egress to GitHub), Kustomizations `zy-tools` and `gpu-operator` Ready. Rollback (DESTRUCTIVE): `kubectl delete -f` (prune removes all children).
- **D2** Watch gpu-operator → run E1. Nothing further until E1 passes. Tool rollback (DESTRUCTIVE): comment out and push; restore `config.toml.pre-nvidia` + restart containerd if needed.
- **D3** Second reboot of the GPU node (D-9), re-run E1: driver rebuilds, toolkit config survives, 4 GPUs return (expect 5–10 min for the driver build).
- **D4** Uncomment `longhorn.yaml`, render, commit, push → E2. Rollback is costly: set Longhorn `deleting-confirmation-flag=true` first, then comment out and push (DESTRUCTIVE); disk data stays.
- **D5** Uncomment `kyverno.yaml` and `kyverno-policies.yaml` together, push → E3. Rollback: policies first, then kyverno. Emergency if the webhook blocks pods: delete Kyverno webhook configurations (DESTRUCTIVE).

### E. Verification (READ-ONLY unless marked)

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

### Risks

1. **Internet from CAPX nodes and Flux is untested.** Only the GPU node was checked. D1/E1 reveal it; if they can't pull, images must go into `registry.ntnxlab.local/library`.
2. **Driver build** needs the Ubuntu archive and repeats on every boot (5–10 min before GPUs are allocatable). Fallbacks if headers can't be fetched (ZY decides): precompiled driver (likely a different version), private apt mirror via `driver.repoConfig`, or driver baked into the node image with `driver.enabled=false`.
3. **Toolkit vs immutable OS:** `/usr/local` persistence (P3) and whether boot rewrites `config.toml` (B5). The toolkit re-applies on start, at the cost of a containerd restart per boot.
4. **Longhorn disk adoption:** `diskName` matching in 1.12 unverified (B2/E2).
5. **Supportability:** upstream gpu-operator v26.7.2; do not also enable NKP's `nvidia-gpu-operator` platform app.
6. **Longhorn uninstall** is not a clean rollback (D4).
7. **Public repo:** if P1 confirms the repo is public, everything committed here (including this plan's IPs and hostnames) is public on push.

---

## Phase 1 progress

Status values: `todo`, `in progress`, `done`, `blocked`, `skipped`.

**Phase 1 status: complete (2026-11-14).** gpu-operator, Longhorn, Kyverno and the nai-admin MutatingPolicies are reconciled by Flux at `e20cbc2`; node advertises nvidia.com/gpu=4. Step F (site facts) done 2026-11-15. Phase 2: see below.

### Open decisions

| # | Question | Status |
|---|---|---|
| D-7 | Longhorn `storageReserved` per disk | Decided 2026-11-14: option A, 50 GiB (53687091200) on disk 1, 0 on disks 2–4 |

### Steps

| Step | What | Mark | Status | Date | Evidence / notes |
|---|---|---|---|---|---|
| 0a | Scan nai-tools for environment-specific values | READ-ONLY | done | 2026-11-14 | Cluster rename already done; public registries/charts, old cloudflared/NAI hostnames (phase 2) |
| 0b | Review and commit pending namespace refactor | CHANGE | done | 2026-11-14 | Commit `171855c` (targetNamespace in tools-flux.yaml); not pushed |
| 0c | Cluster and GPU-node discovery (incl. root read-only checks) | READ-ONLY | done | 2026-11-14 | Kairos node, no headers, disks already Longhorn-initialised, multipath won't claim sda..sdd; see plan "Key facts" |
| P1 | Repo public, charts exist, Kyverno version to pin | READ-ONLY | done | 2026-11-14 | Anonymous `git ls-remote` works (repo **public**, origin/main = `1932943`). gpu-operator v26.7.2, longhorn 1.12.1 exist. Kyverno latest chart 3.10.0 (app v1.20.0), kubeVersion `>=1.25`; official k8s 1.36 support not verified |
| P2 | Flux CRDs, node NFD labels, PSA labels, server dry-run | READ-ONLY | done | 2026-11-14 | HelmRelease v2, others v1. Node: ubuntu 24.04, kernel 6.8.0-134-generic. Privileged hostPath pod server dry-run in `default` accepted with no PSA warning, so no PSA labels needed on new namespaces. `tools-flux.yaml` server dry-run: both objects created, no Gatekeeper denial |
| P3 | GPU node: /oem, /usr/local, Longhorn tools, Ubuntu headers, disks, kubelet unit | READ-ONLY | done | 2026-11-14 | `/oem` (COS_OEM) and `/usr/local` (COS_PERSISTENT) rw and persistent, so default toolkit installDir is fine. iscsiadm, mount.nfs/nfs4, cryptsetup present; iscsid active+enabled; iscsi_tcp/nfs/nfsv4/dm_crypt available. `linux-headers-6.8.0-134-generic` in noble-updates and noble-security. Disk UUIDs distinct (sda1 84a4abc3…, sdb1 e3b548ae…, sdc1 181e1994…, sdd1 f75eb19d…). Unit is `kubelet.service`. /oem yaml uses `stages: initramfs:` |
| A1 | Confirm Longhorn 1.12.1 value keys | READ-ONLY | done | 2026-11-14 | `guaranteedInstanceManagerCPU` exists; Engine/Replica variants absent (confirmed obsolete). createDefaultDiskLabeledNodes, systemManagedComponentsNodeSelector, taintToleration, defaultDataPath, csi.*ReplicaCount, persistence.defaultClassReplicaCount all present. Kyverno 3.10.0: `spec.failurePolicy` deprecated, use `spec.webhookConfiguration.failurePolicy`; `kyverno.io/v1 ClusterPolicy` itself is deprecated (still served) in favour of `policies.kyverno.io` MutatingPolicy |
| A2 | Pin Kyverno chart version | CHANGE | done | 2026-11-14 | `version: "3.10.0"` (Kyverno v1.20.0), ZY's choice |
| A3 | gpu-operator `nfd.enabled: false` (+ installDir if needed) | CHANGE | done | 2026-11-14 | nfd off; installDir unchanged (P3: /usr/local persistent) |
| A4 | Longhorn values: drop old CPU keys, createDefaultDiskLabeledNodes, replica counts 1 | CHANGE | done | 2026-11-14 | |
| A5 | Kyverno policy failurePolicy Ignore | CHANGE | done | 2026-11-14 | `spec.webhookConfiguration.failurePolicy: Ignore` (spec.failurePolicy is deprecated in 3.10.0) |
| A6 | Docs fixes (Longhorn README, SC comment, README step 7) | CHANGE | done | 2026-11-14 | Longhorn README step 1 now has the label/annotation commands incl. storageReserved |
| A7 | flux/kustomization.yaml: gpu-operator only | CHANGE | done | 2026-11-14 | longhorn, kyverno, kyverno-policies commented out |
| A8 | Render check, commit (push later, on ZY's go) | CHANGE | done | 2026-11-14 | Root renders only `gpu-operator`; all 4 overlays render with the new values. Committed locally, not pushed |
| B1 | Mount units var-mnt-longhorn1..4 | CHANGE | done | 2026-11-14 | 4 units enabled (symlinks in local-fs.target.wants). sda1..sdd1 xfs rw on /var/mnt/longhorn1..4, 745G each, 17G/15G/15G/15G used (xfs metadata + ~1.9G data on disk 1) |
| B2 | Read longhorn-disk.cfg UUIDs | READ-ONLY | done | 2026-11-14 | diskName disk1..disk4, diskUUID 9eb58d6c…, cb5f4a76…, a9d2f6d9…, 9b381e67… (distinct), state ready, 0 replicas each. Disk 1 also: nai/ 160M, llmbench/ 1.7G. C1 disk names changed to disk1..disk4 to match the cfg |
| B3 | Multipath blacklist (/oem/95_longhorn.yaml + live) | CHANGE | done | 2026-11-14 | `/oem/95_longhorn.yaml` (0600, initramfs files + boot `multipathd reconfigure`); live `/etc/multipath/conf.d/longhorn.conf` written, `multipathd reconfigure` ok; `multipath -t` blacklist shows `device { vendor "IET" product "VIRTUAL-DISK" }`; multipathd active; `multipath -ll` empty (unchanged) |
| B4 | Back up containerd config.toml | CHANGE | done | 2026-11-14 | `/etc/containerd/config.toml.pre-nvidia` (7483 B, identical to config.toml, mtime 02:20) |
| B5 | Reboot GPU node, verify persistence | CHANGE | done | 2026-11-14 | Reboot 05:26:20 UTC, booted 05:30:32, Ready 05:32:34. 4 xfs mounts back; kubelet `After=` includes all 4 mount units. `/oem/95_longhorn.yaml` ran (immucore initramfs wrote longhorn.conf, kairos-agent boot ran `multipathd reconfigure`); `multipath -t` shows IET blacklist; `multipath -ll` empty. `systemctl --failed` empty; kubelet, containerd, iscsid, multipathd, host-agent active. All 7 pods on the node Running after ~5 min. config.toml content identical to `.pre-nvidia` but mtime 05:32: boot rewrites it (see issues) |
| C1 | Node label + default-disks/default-node-tags annotations | CHANGE | done | 2026-11-14 | Annotations read back: disk1 /var/mnt/longhorn1 reserved 53687091200, disk2..4 reserved 0, all tags [gpu-node]; node-tags ["gpu-node"]; label create-default-disk=config. Re-check for stripping at D1 |
| D1 | Push + bootstrap tools-flux.yaml | CHANGE | done | 2026-11-14 | Pushed `1932943..f178da2`. `kubectl apply -f tools-flux.yaml`: GitRepository `zy-tools` Ready ("stored artifact for revision main@sha1:f178da2…"), proving Flux egress to GitHub; Kustomizations `zy-tools` and `gpu-operator` Ready at f178da2; HelmRepository `nvidia` + HelmRelease `gpu-operator` created |
| D2/E1 | gpu-operator rollout and verification | READ-ONLY | done | 2026-11-14 | HelmRelease Ready ("Helm install succeeded … gpu-operator@v26.7.2") in ~6 min. Operator pod on control-plane node zy-metal-w85rc-7qjs5; driver, toolkit, device-plugin, GFD, DCGM, validator Running on the L40 node; cuda-validator Completed; no NFD pods in gpu-operator. All images from nvcr.io pulled OK (driver 615.71.09-ubuntu24.04, toolkit v1.20.1, device-plugin v0.20.1, dcgm-exporter 4.6.1-4.8.4, driver-manager v0.12.1). Driver log: apt installed linux-headers-6.8.0-134-generic, modules loaded, "Done, now waiting for signal". `nvidia-smi`: 4x NVIDIA L40, 615.71.09, 46068 MiB. Node allocatable nvidia.com/gpu=4, labels gpu.product=NVIDIA-L40, gpu.count=4. Node: config.toml `imports = ["/etc/containerd/conf.d/*.toml"]`, conf.d/99-nvidia.toml present, containerd active. `clusterpolicies.nvidia.com/cluster-policy` ready. Transient warnings during rollout (expected): "no runtime for nvidia is configured" before toolkit finished, containerd.sock refused during toolkit's containerd restart, one driver startup-probe failure while building. All pods on the node Running afterwards |
| D3 | Second GPU-node reboot, re-run E1 | CHANGE | done | 2026-11-14 | Reboot 14:12:41 UTC, booted 14:18:29, node Ready 14:20:45, nvidia.com/gpu=4 + validator ready 14:23:12 (~2.5 min after Ready, ~10.5 min total). From ~14:20:30 to 14:22:28 GPU pods failed with "no runtime for nvidia is configured"; toolkit then rewrote config.toml (7483 → 7551 B) and conf.d/99-nvidia.toml at 14:22:28 and SIGHUP'd containerd; all recovered. `nvidia-smi` 4x L40 615.71.09; cluster-policy ready (briefly notReady until DCGM passed readiness ~14:23:30). Mounts (4) and IET blacklist persisted again |
| D4/E2 | Enable Longhorn, verify, RWX test + cleanup | CHANGE | done | 2026-11-14 | Pushed `5f01f4d` (longhorn enabled), forced reconcile. HelmRelease Ready ("Helm install succeeded … longhorn@1.12.1"); L40 Longhorn node Ready 15:09:37; all longhorn-system pods Running+Ready 15:10:51 (managers restarted a few times during first-minute bootstrap, normal). longhorn-manager on all 8 nodes; instance-manager, engine-image, CSI, driver-deployer only on the L40 node. disk1..4 adopted with existing diskUUIDs, tagged gpu-node, Ready/Schedulable; node tag gpu-node; other 7 nodes `disks={}`. Settings applied as intended. SCs longhorn, longhorn-rwx-gpu, longhorn-static; nutanix-volume still sole default. disk2..4 storageReserved came out 30% (239932033843; 0 in the annotation = "use storage-reserved-percentage-for-default-disk") → patched `nodes.longhorn.io` spec.disks.disk2..4.storageReserved=0, reads back 0. dm_crypt: `modprobe dm_crypt` (in /proc/modules) and added boot-stage `modprobe dm_crypt` to /oem/95_longhorn.yaml; Longhorn condition `KernelModulesLoaded` still False at 15:20 (lastTransition 15:09, not re-evaluated yet; /boot/config has CONFIG_DM_CRYPT=m). `Multipathd` warning accepted. **RWX test passed:** PVC Bound (longhorn-rwx-gpu, 1Gi RWX), pod on L40 node logs `ok`, share-manager on L40 node, volume attached/healthy with 1 replica on disk3 (a9d2f6d9…), replica dir under /var/mnt/longhorn3/replicas, Longhorn iSCSI device `sde` (IET VIRTUAL-DISK) present and `multipath -ll` empty (blacklist works). Test cleanup done (ns, PVC, pod deleted; no Longhorn volumes/replicas left). Restarted longhorn-manager on the L40 node to re-evaluate dm_crypt. Git: `storageReservedPercentageForDefaultDisk: 0` + README note |
| D5/E3 | Enable Kyverno + policies, verify | CHANGE | done | 2026-11-14 | **Final:** pushed `e20cbc2`; Flux (auto-sync) applied it 16:01:17 UTC; `kyverno-policies` Ready and adopted both mpol (READY true). All of zy-tools, gpu-operator, longhorn, kyverno, kyverno-policies Ready at e20cbc2. History: | Pushed `8bc976f`. Kustomization `kyverno` Ready; HelmRelease "Helm install succeeded … kyverno@3.10.0"; admission/background/cleanup/reports controllers Running (all scheduled on the L40 node). Server dry-run pod in `default` admitted unchanged. **Kustomization `kyverno-policies` False:** "admission webhook validate-policy.kyverno.svc denied the request: kyverno.io/v1 ClusterPolicy: Kyverno v1.20 removed execution of legacy kyverno.io policy types and no longer accepts create or update requests for them". `pin-nai-admin-to-l40` not created. ZY chose: migrate to `policies.kyverno.io/v1` MutatingPolicy (plan by planner). New `overlays/kyverno-policies/nai-admin-mutating-policies.yaml` (MutatingPolicy `pin-nai-admin-to-l40`: JSONPatch adds nodeSelector key, keeps existing keys, autogen off; MutatingPolicy `nai-admin-default-pvc-storageclass`: unset/"" → longhorn-rwx-gpu); both CREATE only, `failurePolicy: Ignore`, namespaceSelector `kubernetes.io/metadata.name: nai-admin`. Old ClusterPolicy file removed. Server dry-run OK; applied by hand; both mpol READY true ("Webhook configured"), autogen empty; webhooks `mpol.validate.kyverno.svc-ignore` (pods CREATE) and `...-finegrained-nai-admin-default-pvc-storageclass` (PVC CREATE), both fp=Ignore. Temp ns `nai-admin` dry-runs: T1 pod → `{"node.cluster.x-k8s.io/longhorn-system-managed":"true"}`; T2 pod with disktype=ssd → both keys; T3 PVC unset → `nutanix-volume` (default SC fills it before the webhook, expected; ZY sets the class explicitly in the NAI install); T3b `""` → `longhorn-rwx-gpu`; T4 explicit nutanix-volume unchanged; T5/T6 in `default` unchanged; T7 Deployment template untouched (autogen off). Temp `nai-admin` deleted (was empty). Commit/push + Flux adoption pending |
| F | Record site facts in ops `zy-nkp-site` via doc-writer (Kairos node, containerd 2.3.4, internet vs air-gap) | CHANGE | done | 2026-11-15 | ops `.claude/skills/zy-nkp-site/SKILL.md` updated; new runbook ops `.claude/runbooks/zy-metal-gpu-node.md` (+ index entry). Full Longhorn uninstall with adopted disks marked untested there |

### Decision log

| Date | Decision |
|---|---|
| 2026-11-14 | Deploy from `ops/`; two phases (phase 2 = cloudflared, kubernetes-mcp, openwebui after ZY review) |
| 2026-11-14 | Cluster installed air-gapped but has internet; keep public chart/image sources |
| 2026-11-14 | Keep existing disk data, no reformat |
| 2026-11-14 | gpu-operator `driver.enabled=true` |
| 2026-11-14 | Longhorn replica counts 1; Kyverno policy failurePolicy Ignore; defer nai-admin policy test; second reboot OK |

### Issues found during rollout

- 2026-11-14: `kyverno.io/v1 ClusterPolicy` is deprecated in Kyverno v1.20 (still served, warns on apply). `pin-nai-admin-to-l40` should move to `policies.kyverno.io` MutatingPolicy later; not blocking phase 1.
- 2026-11-14: the GPU node rewrites `/etc/containerd/config.toml` at every boot (content unchanged, mtime updated). Unknown whether it merges or regenerates; if it regenerates, the NVIDIA toolkit's edits are lost each boot and re-added when the toolkit pod starts (extra containerd restart). D3 (second reboot after gpu-operator) shows which. **Result (D3):** boot drops the nvidia runtime (GPU pods saw "no runtime for nvidia" for ~2 min after boot), and the toolkit pod re-adds it and SIGHUPs containerd (no full restart). Self-healing; expect GPUs ~2–3 min after the node is Ready, ~10 min after a reboot. Acceptable; no change.
- 2026-11-14: the repo is public on GitHub; these rollout docs (IPs, hostnames) become public on push.

---

## Phase 2 progress: cloudflared, kubernetes-mcp-server, openwebui

Status values: `todo`, `in progress`, `done`, `blocked`, `skipped`.

### Steps

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

### Decision log

| Date | Decision |
|---|---|
| 2026-11-15 | cloudflared: one tunnel per cluster; routes stay in git |
| 2026-11-15 | k8s-mcp exposed publicly (NAI needs a public-trusted HTTPS URL); protected by Cloudflare Access Bypass limited to the lab egress IP |
| 2026-11-15 | OpenWebUI: no node affinity; signup settings unchanged, ZY registers admin right after deploy |
| 2026-11-15 | Keep the jaeger route for later |
