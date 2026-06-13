# Phase 1 — VirtualBox Foundation
 
> **Goal:** Prepare the hypervisor, installation media, and operating conventions so that all subsequent phases can provision VMs against a consistent baseline. No actual VM provisioning happens in this phase — that begins in Phase 2.
 
## Overview
 
Phase 1 is pure preparation. It produces four deliverables:
 
1. A verified VirtualBox installation with the matching Extension Pack
2. All required installation media downloaded and integrity-checked
3. A folder structure for ISOs and VM storage
4. Documented naming and snapshot conventions used by every subsequent phase
This phase is intentionally boring. Every shortcut taken here multiplies into pain in later phases — corrupted ISOs cause kernel panics that look like driver issues, inconsistent VM names break automation scripts, missing snapshots force full rebuilds when a configuration breaks.
 
---
 
## 1. Host requirements
 
The lab runs on a Windows 11 Pro host with the following specs (per [Phase 0 budget](phase0-planning-design.md#6-hardware--resource-budget)):
 
- Intel Core i7-14700KF (20 cores, 28 threads)
- 32 GB DDR5
- NVMe storage with 250 GB+ free for VM disks
- Hardware virtualization enabled in BIOS (Intel VT-x)
### Verifying hardware virtualization
 
On Windows, hardware virtualization status is reported in Task Manager → Performance → CPU panel, bottom-right: `Virtualization: Enabled`. If disabled, reboot into UEFI/BIOS and enable Intel VT-x (Intel) or SVM Mode (AMD). On HP and Dell systems this setting typically lives under *Advanced* or *Security* submenus.
 
---
 
## 2. VirtualBox + Extension Pack
 
Required: **VirtualBox 7.0 or newer** and the matching Oracle Extension Pack.
 
### Installation
 
VirtualBox installer and matching Extension Pack are downloaded from https://www.virtualbox.org/wiki/Downloads. The Extension Pack version must match the VirtualBox installer version exactly.
 
After installing VirtualBox with default options, the Extension Pack is added via the GUI: **File → Tools → Extension Pack Manager → Install**.
 
### Verification
 
```powershell
VBoxManage --version
VBoxManage list extpacks
```
 
The Extension Pack must appear as `Oracle VM VirtualBox Extension Pack` with `Usable: true`. It provides VRDE (RDP into headless VMs), USB 2.0/3.0 passthrough, and PXE boot for Intel NICs — all used in later phases.
 
---
 
## 3. Storage layout
 
Two dedicated folders are used to keep installation media separate from VM disks:
 
| Purpose | Path | Notes |
|---|---|---|
| ISO storage | `D:\VirtualBox\ISOs\` | All installation media in one place, referenced consistently |
| VM storage | (VirtualBox default machine folder) | Set under File → Preferences → General → Default Machine Folder |
 
The VM storage folder is configured in VirtualBox preferences to point to a partition with at least 250 GB free. With dynamic disk allocation, the eight planned VMs allocate ~303 GB but typically use 130-180 GB in practice.
 
---
 
## 4. Installation media — download and verify
 
Six ISO files are downloaded for the lab. Two ISOs (Ubuntu Server, Windows 11 Pro) are reused across multiple VMs to save bandwidth and storage.
 
| OS | Edition | Source | Approx. size |
|---|---|---|---|
| pfSense CE (via Netgate Installer) | v1.2-RELEASE | https://www.pfsense.org/download/ → Netgate Store | ~330 MB compressed |
| Windows Server 2022 | Eval (180 day) | https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022 | ~5 GB |
| Windows 11 Pro | Latest | https://www.microsoft.com/en-us/software-download/windows11 | ~6 GB |
| Ubuntu Server | 24.04 LTS | https://ubuntu.com/download/server | ~2.5 GB |
| Ubuntu Desktop | 24.04 LTS | https://ubuntu.com/download/desktop | ~5 GB |
| Kali Linux | Installer (latest) | https://www.kali.org/get-kali/#kali-installer-images | ~4 GB |
 
### Decompressing .iso.gz files
 
The pfSense download arrives as `netgate-installer-v1.2-RELEASE-amd64.iso.gz` — a gzip-compressed ISO. VirtualBox cannot mount `.iso.gz` directly, so the file must be decompressed first. On Windows this is done with 7-Zip (right-click → Extract Here) or PowerShell:
 
```powershell
$src = "D:\VirtualBox\ISOs\netgate-installer-v1.2-RELEASE-amd64.iso.gz"
$dst = "D:\VirtualBox\ISOs\netgate-installer-v1.2-RELEASE-amd64.iso"
$in  = [System.IO.File]::OpenRead($src)
$out = [System.IO.File]::Create($dst)
$gz  = New-Object System.IO.Compression.GZipStream($in, [IO.Compression.CompressionMode]::Decompress)
$gz.CopyTo($out); $gz.Close(); $out.Close(); $in.Close()
```
 
### Checksum verification
 
SHA256 checksums are verified against the values published by each project on their official download or checksum page. **Checksums for `.gz` files are computed against the compressed file**, not the extracted ISO — verify before decompressing.
 
```powershell
Get-FileHash -Algorithm SHA256 "D:\VirtualBox\ISOs\*"
```
 
For Microsoft ISOs (Windows Server 2022, Windows 11 Pro) the SHA256 is not always published. In those cases, digital signature verification is used instead: right-click the `.iso` → Properties → Digital Signatures → confirm the signer is `Microsoft Corporation` and the signature is valid.
 
Any checksum mismatch means the file is re-downloaded. No ISO proceeds to the install phase without a verified hash or valid signature.
 
---
 
## 5. VirtualBox preparation
 
A VM Group named **`SOC HomeLab`** is created in VirtualBox to group all eight lab VMs in one collapsible folder. The group exists empty at the end of Phase 1; VMs are added to it as they are created in Phase 2 onward.
 
The four VirtualBox Internal Networks used to model the VLANs are *not* pre-created. They materialize automatically the first time a VM with a matching adapter is started (this happens in Phase 2 when pfSense boots for the first time). The names used are case-sensitive and whitespace-sensitive:
 
| Network name | VLAN | Purpose |
|---|---|---|
| `vlan10-corp` | 10 | Corporate domain |
| `vlan20-dev` | 20 | Development workstations |
| `vlan66-attack` | 66 | Attacker DMZ |
| `vlan99-soc` | 99 | SOC management |
 
---
 
## 6. Conventions
 
### VM naming
 
Format: `SOC-{VLAN-or-zone}-{Role}`. All VMs go inside the `SOC HomeLab` VM Group.
 
| VM | Name |
|---|---|
| pfSense edge router | `SOC-Edge-pfSense` |
| Active Directory DC | `SOC-10-DC` |
| Windows 11 Pro Corporate | `SOC-10-Corp-WS` |
| Windows 11 Pro Dev | `SOC-20-Dev-Win` |
| Ubuntu Desktop Dev | `SOC-20-Dev-Ubuntu` |
| Wazuh manager | `SOC-99-Wazuh` |
| Splunk SIEM | `SOC-99-Splunk` |
| Kali attacker | `SOC-66-Kali` |
 
### Default VM settings
 
Applied to every VM unless overridden in the phase that creates it:
 
| Setting | Value |
|---|---|
| Chipset | ICH9 |
| Audio | Disabled |
| USB Controller | USB 3.0 (xHCI) |
| Shared Clipboard | Bidirectional (after Guest Additions) |
| Drag & drop | Bidirectional |
| Time sync (guest → host) | Enabled |
 
### Snapshot policy
 
Five named snapshot stages are used per VM:
 
| Snapshot name | When to take it |
|---|---|
| `clean-install` | Immediately after OS installation, before any configuration |
| `base-config` | After timezone, hostname, network, and admin user are set |
| `pre-tool` | Before installing the primary tool of that VM (Wazuh, Splunk, AD role, Sysmon, etc.) |
| `working` | After the VM is verified functional |
| `pre-attack` | Before each attack scenario in Phase 9, on the targeted Corp and Dev VMs |
 
Snapshots are reclaimed on delete. The discipline of snapshotting before every destructive operation saves hours of rebuild time over the life of the lab.
 
---
 
## 7. Decisions recorded
 
Two non-obvious decisions were made during Phase 1 and are recorded here for traceability.
 
### 7.1 Windows Server 2022 over 2019
 
**Decision:** Windows Server 2022 Evaluation is used for the Active Directory Domain Controller, not 2019.
 
**Rationale:** The lab is a detection-engineering exercise, not an exploit-development exercise. The attacks targeted (Kerberoasting, AS-REP roasting, password spraying, LSASS dumping, lateral movement via SMB/PsExec) exploit Active Directory protocol behaviour and Windows authentication semantics — not OS-version-specific CVEs. They fire identically on both 2019 and 2022. Detection by Sysmon, Wazuh, and Splunk also behaves identically.
 
What actually matters for "more vulnerable" is the **hardening posture**, not the OS version. The lab uses default Windows Server 2022 configuration without hardening (no LAPS, no tiered admin, no AppLocker, no Credential Guard, no Protected Users group), and adds intentional AD misconfigurations during Phase 4 (Kerberoastable service accounts, AS-REP-eligible users, weak ACLs) to expose the attack surface deliberately.
 
**Trade-off:** None. Server 2022 has mainstream support through October 2026, is the current LTSC release in real enterprises, and reads as a stronger signal in a portfolio than the older 2019.
 
### 7.2 Netgate Installer (new pfSense distribution)
 
**Context:** Netgate (the company behind pfSense) no longer distributes pfSense CE as a standalone ISO. The current distribution is a unified **Netgate Installer** (v1.2-RELEASE at time of build) that handles both pfSense Plus and pfSense CE installs.
 
**How it works:** The installer detects whether the host hardware is eligible for pfSense Plus (Netgate appliances, paid accounts). On non-eligible hardware — including any VirtualBox VM — it presents the option to install pfSense CE.
 
**Implication for the lab:** The download is named `netgate-installer-v1.2-RELEASE-amd64.iso.gz` rather than the classic `pfSense-CE-2.x.x-RELEASE-amd64.iso.gz`. Older tutorials referencing the classic CE ISO are out of date. The actual CE version (2.8.1 at the time of this lab build) is selected during install, not at download time.
 
**Trade-off:** None operationally. The end result is a vanilla pfSense CE install. The only adjustment versus older guides is the artifact name and the extra "select CE" step during installation.
 
---
 
## Next phase
 
[Phase 2 — Network Backbone](phase2-network-backbone.md)
 
