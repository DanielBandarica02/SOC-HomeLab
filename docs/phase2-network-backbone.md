# Phase 2 — Network Backbone (pfSense + Segmentation)
 
> **Goal:** Provision the pfSense edge router as the first VM in the lab, segment the network into four VLANs via VirtualBox Internal Networks, configure routing and NAT, lay down a permissive baseline firewall policy, and install Suricata as the IDS package. By the end of this phase the four Internal Networks materialize and the network is ready to receive every other VM.
 
## Overview
 
Phase 2 produces the network foundation everything else sits on. It covers:
 
1. Provisioning the pfSense VM in VirtualBox with five virtual NICs
2. Installing pfSense CE 2.8.1 via the new Netgate Installer
3. Assigning physical interfaces (em0–em4) to logical roles (WAN, LAN, OPT1, OPT2, OPT3)
4. Configuring static IPs and DHCP servers per VLAN from the console
5. Bootstrapping webGUI access by provisioning the Windows 11 Pro Corporate workstation
6. Running the pfSense Setup Wizard and configuring NAT outbound mode
7. Adding a permissive baseline firewall policy that allows each VLAN outbound while keeping inter-VLAN traffic at default deny on the OPT interfaces
8. Installing Suricata as a pfSense package (the package itself; configuration of monitored interfaces and rule sets happens in Phase 4)
This phase is the most technically dense in the build. Several non-obvious things were learned along the way and are documented in the troubleshooting section.
 
---
 
## 1. pfSense VM provisioning
 
### Hardware specification
 
| Setting | Value |
|---|---|
| Name | `SOC-Edge-pfSense` |
| OS Type | BSD → FreeBSD (64-bit) |
| vCPU | 2 |
| RAM | 2 GB |
| Disk | 8 GB dynamic |
| EFI | Disabled (pfSense boots cleaner with legacy BIOS) |
| Network adapters | 5 (1 NAT + 4 Internal Network) |
 
### Network adapter configuration
 
The pfSense VM has five virtual NICs. Their order matters because pfSense detects them as `em0`, `em1`, `em2`, `em3`, `em4` in that sequence:
 
| Adapter | Attached to | Name | Promiscuous Mode |
|---|---|---|---|
| 1 | NAT | (n/a — VirtualBox NAT) | Deny |
| 2 | Internal Network | `vlan10-corp` | Allow VMs |
| 3 | Internal Network | `vlan20-dev` | Allow VMs |
| 4 | Internal Network | `vlan66-attack` | Allow VMs |
| 5 | Internal Network | `vlan99-soc` | Allow VMs |
 
All five use **Intel PRO/1000 MT Desktop (82540EM)** as adapter type. FreeBSD has native drivers for this NIC.
 
**Promiscuous mode** is set to Allow VMs on the four Internal Network adapters. This lets Suricata (running on pfSense) observe intra-VLAN traffic that does not transit pfSense's routing plane — necessary to detect lateral movement *within* a VLAN, not only across them.
 
---
 
## 2. Interface assignment
 
On first boot pfSense asks how to map the detected physical interfaces (em0–em4) to logical roles. The lab uses this fixed mapping:
 
| Logical role | Physical interface | VirtualBox attachment | Purpose |
|---|---|---|---|
| WAN | em0 | NAT | Internet uplink via VirtualBox NAT |
| LAN | em1 | Internal Network `vlan10-corp` | VLAN 10 — Corporate |
| OPT1 | em2 | Internal Network `vlan20-dev` | VLAN 20 — Development |
| OPT2 | em3 | Internal Network `vlan66-attack` | VLAN 66 — Attacker DMZ |
| OPT3 | em4 | Internal Network `vlan99-soc` | VLAN 99 — SOC Management |
 
Note: pfSense calls VLAN 66 and VLAN 99 by the names `OPT2` and `OPT3` internally. These names are renamed to friendly labels (`ATTACK`, `SOC`) later via the webGUI.
 
VLAN tagging (802.1Q) was declined at the assignment prompt. The lab uses VirtualBox Internal Networks as L2 broadcast domains rather than tagged trunks — this provides functionally equivalent isolation without the complexity of trunk configuration in VirtualBox.
 
---
 
## 4. IP configuration via console
 
Each interface is configured via the pfSense console menu (option `2) Set interface(s) IP address`). The IP plan from Phase 0 is applied:
 
| Interface | IPv4 / mask | DHCP server | DHCP range |
|---|---|---|---|
| WAN | DHCP (assigned by NAT) | n/a | n/a |
| LAN | `10.10.10.1/24` | Enabled | `.100–.200` |
| OPT1 | `10.10.20.1/24` | Enabled | `.100–.200` |
| OPT2 | `10.10.66.1/24` | Disabled | (Kali is static in Phase 6) |
| OPT3 | `10.10.99.1/24` | Disabled | (Wazuh / Splunk are static in Phase 3) |
 
DHCP is enabled on LAN and OPT1 so corporate and dev workstations get IPs automatically. OPT2 (Kali) and OPT3 (SOC servers) are explicitly static — these are infrastructure hosts whose IPs must be predictable for firewall rules and agent configuration.
 
### Web configurator HTTP / HTTPS decision
 
When the console asks `Do you want to revert to HTTP as the webConfigurator protocol?`, the answer is **no** for every interface. HTTPS (default) is kept with the auto-signed certificate — browsers will warn but accept once.
 
A snapshot named `base-config` is taken on the pfSense VM at this point, before any webGUI configuration.
 
---
 
## 5. Bootstrap problem and Win11 Corp workstation
 
After the console configuration completes, pfSense routes traffic between VLANs and out via NAT — but its webGUI is unreachable from outside. The webGUI listens on every LAN/OPT interface (e.g., `https://10.10.10.1` on LAN, `https://10.10.99.1` on SOC), but no VM exists in any of those VLANs yet to send the HTTP request.
 
**This is a deliberate chicken-and-egg:** all subsequent firewall configuration, NAT mode changes, Suricata setup, and password change have to happen in the webGUI, which requires a host inside one of the configured VLANs.
 
The solution is to provision the Windows 11 Pro Corporate workstation (`SOC-10-Corp-WS`) early. It is the most natural bootstrap host because:
 
- VLAN 10 (LAN) has a default-allow firewall rule, so a workstation there immediately gets internet
- DHCP is configured on LAN, so the VM gets an IP at first boot without static configuration
- It is needed anyway as part of the corporate environment (Phase 4)
The VM is created in VirtualBox per the spec sheet from Phase 1 (4 GB RAM, 2 vCPU, 50 GB disk, EFI + TPM 2.0 + Secure Boot enabled for Windows 11 compatibility). Windows 11 is installed off the official ISO. During the OOBE the Microsoft account requirement is bypassed by disconnecting the virtual network cable (Devices → Network → uncheck "Connect Network Adapter") at the "Let's connect you to a network" screen. After the local account is created the cable is reconnected.
 
Once Windows 11 Corp is booted with an IP from pfSense's DHCP (e.g., `10.10.10.102` with gateway `10.10.10.1`), `https://10.10.10.1` becomes accessible. The remainder of Phase 2 is configured from this workstation.
 
A snapshot `clean-install` is taken on the Win11 Corp VM at this point. All further Win11 Corp configuration (domain join, Sysmon, Wazuh agent) is deferred to Phase 4.
 
---
 
## 6. pfSense Setup Wizard (webGUI)
 
The Setup Wizard runs at first webGUI login. Key settings:
 
| Step | Setting | Value |
|---|---|---|
| General Information | Hostname | `soc-pfsense` |
| | Domain | `soclab.local` |
| | Primary DNS | `1.1.1.1` |
| | Secondary DNS | `8.8.8.8` |
| | Override DNS | unchecked |
| Time Server | Time server | `2.pfsense.pool.ntp.org` |
| | Timezone | `Europe/Madrid` |
| WAN Configuration | Type | DHCP |
| | Block RFC1918 Private Networks | **unchecked** |
| | Block bogon networks | checked |
| LAN Configuration | IPv4 | `10.10.10.1/24` (already set via console) |
| Admin password | New password | (chosen, not stored in repo) |
 
**Block RFC1918 private networks** is explicitly unchecked because the VirtualBox NAT places pfSense's WAN inside the private range `10.0.2.0/24`. With the default "block private" setting enabled, pfSense would block its own gateway — leading to no internet at all from the lab.
 
The default admin password `pfsense` is changed during the wizard. No other administrative account is created at this stage.
 
---
 
## 7. NAT outbound mode → Hybrid
 
After the wizard, NAT outbound mode is changed from **Automatic** to **Hybrid Outbound NAT** under `Firewall → NAT → Outbound`. Hybrid mode keeps the auto-generated NAT rules and lets manual rules be added alongside.
 
Why Hybrid rather than Manual: Hybrid retains the safe defaults (each VLAN automatically gets a NAT outbound rule to WAN). Manual mode requires recreating those rules by hand, which is a recipe for missing one and breaking outbound for a VLAN.
 
---
 
## 8. Firewall rules — permissive baseline
 
The OPT1, OPT2, and OPT3 interfaces have *no default rules* and therefore deny everything by default. To bootstrap subsequent VMs (Wazuh and Splunk Ubuntu Server installs need internet for `apt`, Kali needs internet for tool updates, etc.), a permissive baseline is added per OPT interface:
 
| Interface | Action | Source | Destination | Description |
|---|---|---|---|---|
| OPT1 | Pass | `OPT1 subnets` | any | `Allow OPT1 (Development) to any — bootstrap permissive, tightened in Phase 8` |
| OPT2 | Pass | `OPT2 subnets` | any | `Allow OPT2 (Attacker DMZ) to any — bootstrap permissive, tightened in Phase 8` |
| OPT3 | Pass | `OPT3 subnets` | any | `Allow OPT3 (SOC Management) to any — bootstrap permissive, tightened in Phase 8` |
 
The LAN interface keeps its default Anti-Lockout Rule plus the auto-generated "Default allow LAN to any" rule — no changes there.
 
This baseline is deliberately permissive for two reasons:
 
1. **It lets the lab build proceed.** Without these rules, every VM in VLAN 20/66/99 fails to install operating system packages, agents, or anything that needs internet.
2. **Strict policy needs validation against a working lab.** The actual inter-VLAN deny rules require knowing which agent connections need to traverse which boundaries (Wazuh agents from corp/dev pushing to SOC, etc.). Restricting now means breaking later.
The tightening happens in the session immediately before Phase 8, when all eight VMs are running and the strict policy can be validated against actual telemetry flows.
 
The description on every rule explicitly says "bootstrap permissive, tightened in Phase 8" — this signals deliberate decision rather than oversight when reviewed in interviews.
 
### pfSense 2.8.1 source/destination naming
 
In pfSense 2.8.1 the source/destination dropdown renames `OPT1 net` (from older versions) to **`OPT1 subnets`**. Same meaning — the entire subnet behind the interface — just renamed. Tutorials written for older pfSense will reference `OPT1 net`; the equivalent is now `OPT1 subnets`.
 
---
 
## 9. Suricata package installation
 
Suricata is installed via `System → Package Manager → Available Packages`. The package listing shows:
 
- Package name: `suricata`
- Version: `7.0.8_5` (pfSense package wrapper revision)
- Dependency: `suricata-7.0.1` (FreeBSD upstream port — the actual Suricata binary)
The package + its dependency install automatically. After install completes, `Services → Suricata` appears in the menu — empty (no interfaces configured, no rule sets enabled).
 
Suricata is intentionally left unconfigured here. Its actual setup — assigning to LAN and OPT interfaces, enabling ET Open rules, configuring `eve.json` output, integrating with Wazuh and Splunk HEC — is part of Phase 4, after the SOC stack exists in Phase 3 to ingest the alerts.
 
After install, a snapshot named `working` is taken on the pfSense VM.
 
---
 
## 10. Troubleshooting log
 
Real problems encountered during this phase, with the diagnostic process and resolution for each. These are kept in the doc both as a reference for future rebuilds and as material for portfolio narrative.
 
### 10.1 "CPU doesn't support long mode" on pfSense boot
 
**Symptom:** the pfSense kernel aborted during the first boot of the installer with the message `CPU doesn't support long mode`. Boot looped back to the loader prompt.
 
**Diagnostic:** the host CPU (Intel i7-14700KF) absolutely supports x86-64 long mode, so the issue had to be in how VirtualBox was exposing CPU features to the guest.
 
**Root cause:** the VM was configured with OS Type **FreeBSD** (32-bit) rather than **FreeBSD (64-bit)**. The Netgate Installer ISO is amd64-only and refuses to boot on a CPU presented as 32-bit.
 
**Fix:** Settings → General → Basic → Version → **FreeBSD (64-bit)**. Boot resumed normally.
 
**Lesson:** the VirtualBox OS Type dropdown lists both 32-bit and 64-bit variants of every guest. Always select the 64-bit variant explicitly for modern OS installs.
 
### 10.2 pfSense 2.8.1 console asks for DHCP yes/no before static IP
 
**Symptom:** during console IP configuration for LAN, the first prompt was `Configure IPv4 address LAN interface via DHCP? (y/n)`. Older tutorials and guides describe the prompt as `Enter the new LAN IPv4 address` directly. Typing an IP address at the DHCP-yes/no prompt caused the prompt to repeat (input not understood).
 
**Diagnostic:** the prompts in pfSense 2.8.1 are reordered compared to older releases.
 
**Root cause:** in 2.8.x the DHCP yes/no question comes first. Only after answering `n` (no) does the wizard ask for a static IPv4 address.
 
**Fix:** for LAN/OPT interfaces that should be static, answer **`n`** at the DHCP prompt, then the wizard asks for IPv4 address, subnet bit count, gateway (press Enter for LAN/OPTs), IPv6, DHCP server settings, etc.
 
**Lesson:** pfSense version differences matter for documentation — guides written against 2.6.x or 2.7.x will not match 2.8.x prompt sequences word-for-word.
 
### 10.3 Suricata install fails — DNS not resolving on pfSense
 
**Symptom:** the Suricata package install from the webGUI failed to fetch the package. From the pfSense console shell:
 
```
# pkg install pfSense-pkg-suricata
pkg: Error fetching ...: name resolution failure
```
 
**Diagnostic:** the methodology used to isolate the problem was the classic "is it the network or is it DNS" split, using `Diagnostics → Ping` in the pfSense webGUI:
 
1. **Ping by IP** — target `8.8.8.8` → **success**, replies under 30 ms. This confirmed IP-layer connectivity to the internet via NAT was healthy. The lab's WAN, NAT outbound rules, and firewall state were correct.
2. **Ping by hostname** — target `google.com` → **failure**, "host not found". DNS resolution was broken on pfSense itself.
The split-test (IP success + DNS failure) localized the problem to pfSense's DNS Resolver, not connectivity.
 
**Root cause:** the DNS Resolver (Unbound) was running with **DNSSEC Support enabled**. With DNSSEC enabled, Unbound validates DNS responses against signed DNSKEY records. The DNS path through VirtualBox's NAT (which proxies DNS via the host's resolver) does not consistently forward DNSSEC chain data, causing validation to fail and queries to return `SERVFAIL`. The result is that DNS-by-name fails for every external lookup, including the pkg repository for Suricata.
 
**Fix:** `Services → DNS Resolver → General Settings` → uncheck **Enable DNSSEC Support** → Save → Apply Changes. DNS resolution resumed working.
 
[Screenshot: `Diagnostics → Ping → 8.8.8.8 succeeds`]
[Screenshot: `Diagnostics → Ping → google.com fails before fix`]
[Screenshot: `Services → DNS Resolver → DNSSEC unchecked`]
[Screenshot: `Diagnostics → Ping → google.com succeeds after fix`]
[Screenshot: `Package Manager → Suricata install succeeds`]
 
**Lesson:** DNSSEC validation is correct security practice in production environments where the upstream resolver chain is trustworthy. In a lab where DNS is proxied through layers (host → VirtualBox NAT → ISP resolver), DNSSEC failure modes are common. For this lab, the trade-off favours availability over signature validation — DNSSEC is disabled deliberately. In production environments served by proper DNSSEC-aware upstream resolvers, this would be re-enabled.
 
### 10.4 Windows 11 OOBE — bypassing the Microsoft account requirement
 
**Symptom:** Windows 11 Setup reached the "Let's connect you to a network" screen and would not allow a local account — only Microsoft account sign-in.
 
**Diagnostic:** the classic `OOBE\BYPASSNRO` command (via Shift+F10) was removed in Windows 11 24H2.
 
**Root cause:** Microsoft progressively closes off the "local account at install time" paths in newer Windows 11 builds.
 
**Fix:** the most reliable cross-version workaround is to **disconnect the virtual network cable** before reaching that screen. In VirtualBox: `Devices → Network → uncheck "Connect Network Adapter"`. The OOBE detects no internet and offers the "Continue with limited setup" path which permits local account creation. The cable is reconnected after the OOBE completes.
 
**Lesson:** Microsoft's policy on local accounts is a moving target. Cable-disconnect is hardware-level — it works regardless of which OOBE bypass commands Microsoft removes.
 
### 10.5 VirtualBox GUI exposes only 4 NIC tabs
 
**Symptom:** the pfSense VM requires five network adapters (1 NAT + 4 Internal Networks) but the Network settings panel in VirtualBox only shows four adapter tabs.
 
**Diagnostic:** verified the VirtualBox limit is 8 NICs per VM via `VBoxManage --help modifyvm`. The GUI was the constraint, not the underlying VM.
 
**Fix:** the fifth adapter is configured via CLI:
 
```powershell
VBoxManage modifyvm "SOC-Edge-pfSense" --nic5 intnet --intnet5 "vlan99-soc" `
  --nictype5 82540EM --nicpromisc5 allow-vms --cableconnected5 on
```
 
The adapter is fully functional after this, even though the GUI continues to show only the first four. Inspection of the full NIC list is done via `VBoxManage showvminfo`.
 
**Lesson:** when VirtualBox's GUI hits a limit, the CLI usually has the full feature. `VBoxManage` is the source of truth for VM configuration; the GUI is a partial view.
 
---
 
## 11. Decisions recorded
 
### 11.1 VirtualBox Internal Networks vs real 802.1Q VLANs
 
**Decision:** VLANs are modeled as VirtualBox Internal Networks (one per VLAN) rather than 802.1Q tagged trunks on a single virtual switch.
 
**Rationale:** Internal Networks provide L2 broadcast domain isolation, which is functionally equivalent to L3 isolation for routing/firewall purposes. The pfSense VM has one NIC per VLAN; pfSense treats each NIC as a separate L3 interface. From the perspective of routing, NAT, and firewall policy, the behavior is indistinguishable from real VLAN trunks.
 
**Trade-off:** VLAN hopping attacks (double-tagging, native VLAN abuse, Yersinia) cannot be practiced — those require real 802.1Q infrastructure. For a detection-engineering lab focused on L3+ TTPs, this is an acceptable trade-off. A future iteration of the lab on Proxmox VE could add real VLAN-aware switches if needed.
 
### 11.2 Suricata on pfSense (package), not as a separate VM
 
**Decision:** Suricata runs as a pfSense package in IDS mode, not as a separate Linux VM with a SPAN/mirror port.
 
**Rationale:** lower RAM cost (no extra VM), simpler topology, and Suricata sees all traffic at pfSense's LAN interfaces — which is where inter-VLAN routing happens. Promiscuous mode on the Internal Network adapters lets Suricata also see intra-VLAN traffic.
 
**Trade-off:** running Suricata inline as IPS (blocking) carries availability risk in a lab — a misfire could break everything. IDS mode (alert-only) is the safer default and what real SOCs typically run for new detection rules.
 
### 11.3 Permissive firewall baseline, tightened later
 
**Decision:** OPT1, OPT2, and OPT3 have "allow any to any" rules during initial build. Strict inter-VLAN policy is added in the session immediately before Phase 8.
 
**Rationale:** strict policy requires knowing which agent flows need which boundaries open (Wazuh agent push paths, Splunk HEC, DNS to internal resolver, NTP, etc.). Locking down before those flows are known means iterating on broken state. Locking down after means iterating on observed state with traffic samples to test against.
 
**Trade-off:** the lab is permissive during Phase 3–7. For portfolio purposes this is explicitly described as a deliberate sequencing choice, not security oversight. Every rule's description includes "bootstrap permissive, tightened in Phase 8" so the choice is unambiguous to anyone reviewing the config.
 
### 11.4 DNSSEC disabled on the DNS Resolver
 
**Decision:** Unbound's DNSSEC validation is disabled in `Services → DNS Resolver → General Settings`.
 
**Rationale:** the lab's WAN path traverses VirtualBox NAT, which proxies DNS through the host OS resolver. DNSSEC chain delivery via this path is unreliable, producing `SERVFAIL` for legitimately signed zones and breaking name resolution lab-wide.
 
**Trade-off:** signature validation is forfeited. The lab is not production. The trade is between availability (lab works) and a security property that does not meaningfully apply at lab scope. In production, DNSSEC stays on and the upstream resolver chain is engineered to support it.
 
---
 
## Next phase
 
[Phase 3 — SOC Stack Deployment](phase3-soc-stack.md)
