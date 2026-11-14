# zy-metal phase 1 progress

Plan: [phase1-plan.md](phase1-plan.md). Update this file after every step: status, date, and the key evidence line.

Status values: `todo`, `in progress`, `done`, `blocked`, `skipped`.

## Open decisions

| # | Question | Status |
|---|---|---|
| D-7 | Longhorn `storageReserved` per disk | Decided 2026-11-14: option A, 50 GiB (53687091200) on disk 1, 0 on disks 2–4 |

## Steps

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
| D1 | Push + bootstrap tools-flux.yaml | CHANGE | todo | | |
| D2/E1 | gpu-operator rollout and verification | READ-ONLY | todo | | |
| D3 | Second GPU-node reboot, re-run E1 | CHANGE | todo | | |
| D4/E2 | Enable Longhorn, verify, RWX test + cleanup | CHANGE | todo | | |
| D5/E3 | Enable Kyverno + policies, verify | CHANGE | todo | | |
| F | Record site facts in ops `zy-nkp-site` via doc-writer (Kairos node, containerd 2.3.4, internet vs air-gap) | CHANGE | todo | | |

## Decision log

| Date | Decision |
|---|---|
| 2026-11-14 | Deploy from `ops/`; two phases (phase 2 = cloudflared, kubernetes-mcp, openwebui after ZY review) |
| 2026-11-14 | Cluster installed air-gapped but has internet; keep public chart/image sources |
| 2026-11-14 | Keep existing disk data, no reformat |
| 2026-11-14 | gpu-operator `driver.enabled=true` |
| 2026-11-14 | Longhorn replica counts 1; Kyverno policy failurePolicy Ignore; defer nai-admin policy test; second reboot OK |

## Issues found during rollout

- 2026-11-14: `kyverno.io/v1 ClusterPolicy` is deprecated in Kyverno v1.20 (still served, warns on apply). `pin-nai-admin-to-l40` should move to `policies.kyverno.io` MutatingPolicy later; not blocking phase 1.
- 2026-11-14: the GPU node rewrites `/etc/containerd/config.toml` at every boot (content unchanged, mtime updated). Unknown whether it merges or regenerates; if it regenerates, the NVIDIA toolkit's edits are lost each boot and re-added when the toolkit pod starts (extra containerd restart). D3 (second reboot after gpu-operator) shows which.
- 2026-11-14: the repo is public on GitHub; these rollout docs (IPs, hostnames) become public on push.
