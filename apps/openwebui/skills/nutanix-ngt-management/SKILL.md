---
name: nutanix-ngt-management
description: Audit, install and uninstall Nutanix Guest Tools (NGT) on AHV VMs through the Nutanix v4 MCP server (vmm namespace), with in-guest steps for both Linux and Windows. Use this whenever the user asks which VMs have or lack NGT, wants an NGT compliance or coverage report, wants to install, mount, enable, upgrade, remove or uninstall NGT, mentions guest tools, NGT ISO, VSS snapshots, self-service restore, or an NGT "not reachable" problem on a Nutanix / Prism Central / AHV cluster — even if they just say "guest tools".
---

# Nutanix Guest Tools (NGT) management

All cluster-side calls go through the Nutanix MCP `vmm_execute` tool. Call
`getOperationSchema` once per operation before the first execute in a session
(the server requires it). Every write needs the `If-Match` header set to the
`_etag` from a GET of the same resource done just before it — always re-GET to
refresh the etag after any write, or the next write fails with 412.

Operations used:

| Purpose | Operation | Notes |
|---|---|---|
| List VMs + NGT state | `ahv_listVms` | GET, paginated, max `$limit` 100 |
| NGT state of one VM | `getGuestToolsById` | path `extId` = VM extId; returns `_etag` |
| CD-ROMs of a VM | `listCdRomsByVmId` | path `vmExtId` |
| Mount NGT ISO | `insertVmGuestTools` | POST, `If-Match`, body below |
| Remote install | `installVmGuestTools` | needs guest OS credentials — see Install |
| Uninstall | `uninstallVmGuestTools` | POST, `If-Match`, no body |
| Upgrade | `upgradeVmGuestTools` | POST, `If-Match` |
| Eject a CD-ROM | `ejectCdRomById` | path `vmExtId` + `extId` |

## Safety rules

- Read-only audit calls need no confirmation. Every write (insert ISO, install,
  upgrade, uninstall, eject CD-ROM, power actions) needs an explicit yes from
  the user for that VM in this conversation, naming the VM.
- Never type or pass guest OS usernames/passwords into `installVmGuestTools`.
  Use the ISO-mount path and let the user run the installer in the guest, or
  have them run the remote install themselves from Prism Central.
- Install and uninstall can reboot the guest; say so when asking.

## 1. Audit: which VMs have / don't have NGT

`guestTools/isInstalled` is **not** an OData filter field, so list everything
and classify client-side:

```json
operation: ahv_listVms
query_params: {
  "$limit": 100, "$page": 0, "$orderby": "name",
  "$select": "extId,name,powerState,cluster/extId,isAgentVm,guestTools/isInstalled,guestTools/isEnabled,guestTools/isReachable,guestTools/version,guestTools/isIsoInserted"
}
```

If `metadata.totalAvailableResults` > 100, page with `$page` 1, 2, … until all
are collected. Confirm the count matches `totalAvailableResults`.

Classify each VM:

| What the response shows | Meaning |
|---|---|
| No `guestTools` block at all | NGT never enabled / not installed |
| `isEnabled: true`, `isInstalled: false` | Enabled (often ISO mounted earlier) but installer never ran |
| `isInstalled: true`, `isReachable: false` | Installed but agent can't talk to the CVM (see Troubleshooting) |
| `isInstalled: true`, `isReachable: true` | Healthy |
| `version` lower than `availableVersion` (from `getGuestToolsById`) | Upgrade available |

A missing `guestTools` block is normal for VMs that never had NGT. To be sure,
spot-check one with `getGuestToolsById` — an empty data object confirms it.

Report as a table grouped by role, and call out which ones actually need NGT:

- **Skip (Nutanix system appliances):** Prism Central VMs (`NTNX-*-PCVM-*`),
  Objects VMs (`objects-*`), Files FSVMs, `isAgentVm: true`. NGT is not expected
  on these.
- **Usually skip (immutable K8s nodes):** NKP / CAPI nodes (names like
  `<cluster>-<hash>-<hash>`, `*-md-0-*`, `*-cp-*`, `*-wn-*`). Hand installs are
  lost when nodes are replaced; if NGT is needed, bake it into the node image.
- **Real gaps:** Windows servers (especially domain controllers and databases,
  where VSS-consistent snapshots matter), bastions, utility and app VMs.
- Note powered-off VMs separately — they must be on to install.

## 2. Install NGT

### 2a. Check the VM is ready

1. `getGuestToolsById` → note `isInstalled`, `isIsoInserted`, `availableVersion`, `_etag`.
2. VM must be powered on.
3. `listCdRomsByVmId` → there must be a CD-ROM whose `backingInfo` is empty.
   If the only drive holds an ISO (commonly `isoType: GUEST_CUSTOMIZATION`, the
   cloud-init / sysprep seed), offer the user two choices:
   - Eject it (`ejectCdRomById`, no downtime). Safe once first-boot
     customization has run, but the seed can't easily be re-attached.
   - Add a second CD-ROM (`createCdRom`) — on AHV this usually requires the VM
     to be powered off first.
   If you skip this check, `insertVmGuestTools` returns a task ID and updates
   capabilities but the ISO silently doesn't mount (`isIsoInserted` stays false).

### 2b. Mount the NGT ISO

```json
operation: insertVmGuestTools
path_params: { "extId": "<vm extId>" }
headers: { "If-Match": "<_etag from getGuestToolsById>" }
request_body: { "capabilities": ["VSS_SNAPSHOT"], "isConfigOnly": false }
```

Capabilities: `VSS_SNAPSHOT` for app-consistent snapshots (Windows VSS; on
Linux it enables pre/post freeze scripts). Add `SELF_SERVICE_RESTORE` for
Windows VMs if the user wants in-guest file restore. Re-GET and confirm
`isIsoInserted: true`.

### 2c. Run the installer inside the guest (user does this)

**Linux** (needs Python 3):
```bash
sudo mkdir -p /mnt/ngt
sudo mount -o ro /dev/disk/by-label/NUTANIX_TOOLS /mnt/ngt   # or /dev/sr0; check lsblk
sudo python3 /mnt/ngt/installer/linux/install_ngt.py
```

**Windows** (elevated PowerShell or cmd), interactive: open the
`NUTANIX_TOOLS` drive and run `setup.exe`. Silent:
```powershell
$d = (Get-Volume -FileSystemLabel NUTANIX_TOOLS).DriveLetter
& "${d}:\setup.exe" /quiet ACCEPTEULA=yes /norestart
```
Windows may want a reboot to finish driver (VirtIO / mobility) updates.

### 2d. Verify

Within a couple of minutes `getGuestToolsById` should show `isInstalled: true`,
`isReachable: true`, a `version`, and the ISO auto-ejected. If not, see
Troubleshooting.

### Remote install alternative

`installVmGuestTools` (body: `capabilities`, `credential{username,password}`,
`rebootPreference{scheduleType: SKIP|IMMEDIATE|LATER}`) logs into the guest and
runs the installer for you. Because it needs the guest password, Claude does
not run it; tell the user they can trigger it from Prism Central (VM → Actions
→ Install NGT) if they prefer no manual guest step.

## 3. Uninstall NGT

### Preferred: API (when NGT is installed and reachable)

1. `getGuestToolsById` → confirm `isInstalled: true` and `isReachable: true`; take `_etag`.
2. Confirm with the user (may reboot the guest).
3. ```json
   operation: uninstallVmGuestTools
   path_params: { "extId": "<vm extId>" }
   headers: { "If-Match": "<_etag>" }
   ```
4. Re-GET and confirm `isInstalled: false`.

### Fallback: inside the guest (when NGT is unreachable)

**Linux:**
```bash
sudo find /usr/local/nutanix -name uninstall_ngt.py   # locate the script
sudo python3 /usr/local/nutanix/ngt/python/bin/uninstall_ngt.py
```
(Use whatever path `find` returned if it differs.)

**Windows:** Settings → Apps → *Nutanix Guest Tools* → Uninstall, or:
```powershell
Get-Package -Name "Nutanix Guest Tools*" | Uninstall-Package -Force
```
Leave the *Nutanix VirtIO* package installed — removing it can break disk and
network drivers.

After an in-guest uninstall, the cluster may still show NGT as enabled; that is
harmless, and the next audit will show `isInstalled: false`.

## 4. Upgrade

If `version` < `availableVersion` and the VM is reachable, `upgradeVmGuestTools`
(with `If-Match`) upgrades in place — check its schema first for the
`rebootPreference` body. Confirm with the user before running.

## Troubleshooting `isReachable: false`

- Guest firewall / network must allow outbound TCP **2074** to the cluster
  virtual IP (Prism Element VIP). Without a cluster VIP configured, NGT can't connect.
- Service running? Linux: `sudo systemctl status ngt_guest_agent`.
  Windows: `Get-Service "Nutanix Guest Agent"`.
- VM was cloned from a template that already had NGT: the cloned NGT identity
  is stale. Re-run `insertVmGuestTools` on the clone to regenerate it, then
  restart the agent.
- Recently installed: give it 2–3 minutes before re-checking.
