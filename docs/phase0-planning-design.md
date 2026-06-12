# Phase 0 — Planning & Design
 
> **Goal:** Define the architecture, scope, and design decisions for the lab before any infrastructure is built. This document is the spec that subsequent phases follow.
 
## Overview
 
This phase produces no infrastructure. It consolidates the threat model, IP plan, hardware budget, and the rationale behind every architectural choice. Anyone reading this document should understand *why* the lab is structured the way it is — not just *what* gets built.
 
The lab is intentionally over-designed for a home lab and under-scoped relative to a real SOC. Both choices are documented below.
 
---
 
## 1. Objectives
 
The lab supports four concrete outcomes:
 
1. **Build a production-grade SOC** in miniature, with realistic network segmentation, telemetry pipelines, and out-of-band management.
2. **Develop Blue Team skills** end-to-end: detection engineering, alert triage, incident response, and forensic investigation.
3. **Practice the purple team workflow** by attacking the lab from the included Kali host and validating detections against generated events.
4. **Demonstrate the above** in a portfolio repository structured for a Junior SOC Analyst hiring process.
---
 
## 2. Scope
 
### In scope (Phases 1–10)
 
- Eight virtual machines across four VLANs
- pfSense routing, firewall, and OpenVPN
- Suricata IDS on perimeter LAN interfaces
- Wazuh EDR (manager + agents on all endpoints)
- Splunk Enterprise SIEM with HEC ingestion
- Active Directory domain in the corporate VLAN
- Sysmon and Auditd telemetry on endpoints
- 15+ custom detection rules mapped to MITRE ATT&CK
- Four end-to-end attack scenarios with incident response reports

---
 
## 3. Threat Model
 
The lab models a small enterprise with an internet-exposed perimeter, internal corporate users, an isolated development network, and a SOC team that monitors but does not actively administer endpoints.
 
### Threats simulated
 
- External attacker performing reconnaissance, exploitation, and lateral movement (Kali in VLAN 66 via NAT)
- Credential theft: password spraying, Kerberoasting, LSASS memory access
- Privilege escalation post-compromise
- Lateral movement using legitimate protocols (RDP, SMB, WinRM)
- Persistence via scheduled tasks, registry Run keys, new local accounts
- Data exfiltration via HTTPS and DNS tunneling
- Ransomware behavior — file rename + shadow copy deletion, simulated only
  
### Threats explicitly not simulated
 
- Real malware execution — all impact-stage activity uses Atomic Red Team tests or custom safe scripts
- Volumetric DoS attacks — out of scope for detection engineering practice
- Supply chain compromises
- Cloud account takeover
- Insider threats holding legitimate administrative credentials by policy (only compromised insiders are modeled)
- 
### Trust boundaries
 
| Boundary | Direction | Enforcement |
|---|---|---|
| Internet ↔ Corporate | Inbound denied (no port forwards) | pfSense WAN rules |
| VLAN 10 ↔ VLAN 20 | Corp → Dev only via VPN/RDP | pfSense LAN rules + OpenVPN |
| VLAN 99 ↔ data plane | Push-only inbound (TCP 1514, TCP 8088) | pfSense LAN rules |
| VLAN 66 ↔ anything | Default deny; attack scenarios open scenario-specific rules | pfSense LAN rules |
 
---
 
## 4. Architecture Decisions
 
Each decision is recorded with rationale, trade-offs, and alternatives considered. Future revisions of the lab can challenge or revise these — but the reasoning is preserved.
 
### 4.1 Network segmentation via VLANs
 
**Decision:** Four isolated VLANs (10 Corporate, 20 Development, 66 Attacker, 99 SOC) routed through pfSense.
 
**Rationale:** Models realistic enterprise segmentation. Enables non-trivial firewall policy and inter-VLAN lateral movement detection scenarios. Without segmentation, the problem space collapses to single-subnet packet inspection, which is not how modern SOCs operate.
 
**Alternative considered:** Single flat network at `192.168.10.0/24`
**Why rejected:** Eliminates the most important detection problem space — lateral movement across trust zones.
 
**Implementation note:** VirtualBox Internal Networks are used as virtual broadcast domains, one per VLAN. This is functionally equivalent to L3 isolation but is *not* real 802.1Q tag-based VLAN trunking. VLAN hopping attacks (Yersinia, double-tagging) cannot be practiced in this setup.
 
### 4.2 pfSense as the routing and firewall plane
 
**Decision:** pfSense CE on a dedicated VM with five interfaces (WAN + four LAN, one per VLAN).
 
**Rationale:** Free, well-documented, supports L3 inter-VLAN routing, has a mature package ecosystem (Suricata, OpenVPN, pfBlockerNG), and matches real-world SMB deployment patterns. Easier to demonstrate in a portfolio than `iptables` scripts.
 
**Alternatives considered:**
- **OPNsense** — functionally equivalent; pfSense chosen for documentation breadth.
- **Linux router with `nftables`** — more control, more configuration overhead, less recognizable in interviews.
- **Cisco vIOS / similar** — licensing complications.
### 4.3 Suricata IDS on pfSense, not as a separate VM
 
**Decision:** Suricata runs as a pfSense package in IDS (passive) mode, monitoring all LAN interfaces.
 
**Rationale:** Single VM, lower RAM cost (Suricata shares pfSense resources), sees all inter-VLAN traffic at the routing point. In a 32 GB RAM budget this is the only economical choice.
 
**Trade-off:** Cannot run reliably in IPS (inline blocking) mode without risking lab availability. Accept IDS-only — detections fire but traffic is not dropped.
 
**Alternative considered:** Separate Suricata VM with a port mirror from pfSense LAN switch.
**Why rejected:** Adds 4 GB RAM cost and significant configuration complexity (span ports in VirtualBox are non-trivial). Can revisit if the host is upgraded to 64 GB.
 
### 4.4 Out-of-band SOC management plane
 
**Decision:** Wazuh and Splunk live in VLAN 99 with strict push-only policy. Agents on data-plane hosts establish outbound TCP connections to the manager; the SOC VLAN never initiates connections into corporate or development.
 
**Rationale:** An attacker who compromises a corporate workstation cannot pivot into the monitoring infrastructure. If the SOC VLAN were reachable, the attacker could disable agents, tamper with logs, or pivot through the dashboard.
 
**Trade-off:** Slightly asymmetric firewall rules. Management of Wazuh and Splunk requires connecting from a host with explicit allowance — in practice, a tightly-scoped jump rule from the analyst's primary workstation.
 
**Real-world parallel:** Standard SOC architecture; aligns with NIST SP 800-92 §4.3 (log management infrastructure isolation).
 
### 4.5 Dedicated attacker DMZ with NAT-simulated external access
 
**Decision:** Kali in VLAN 66, behind pfSense NAT. All "external" attacks originate from this VLAN and traverse pfSense (and Suricata) to reach data-plane VLANs.
 
**Rationale:** Forces realistic perimeter detection paths — Suricata edge alerts, NAT-translated source IPs, geolocation anomalies (if GeoIP rules are added), unauthorized RDP attempts. The lazy alternative ("Kali on the corporate VLAN") trivializes the problem.
 
**Trade-off:** Routing setup is more involved. Some attacks (DNS tunneling exfiltration) need extra plumbing to exit through pfSense's NAT.
 
### 4.6 VirtualBox hypervisor
 
**Decision:** VirtualBox 7.x on a Windows host. Each VLAN modeled as a VirtualBox Internal Network.
 
**Rationale:** Free, available, runs on the existing workstation OS without dual-boot. Continuity with the prior lab iteration's tooling.
 
**Trade-off:** Internal Networks are L2 broadcast domains, not 802.1Q tagged VLANs. This is fine for L3 isolation but cannot teach VLAN-layer attacks.
 
**Future option:** Proxmox VE on dual-boot would provide real 802.1Q trunking, better KVM-based performance, and bridge-VLAN-aware switching — at the cost of dedicating the workstation to lab work.
 
---
 
## 5. IP Addressing Plan
 
All VLANs use the `10.10.0.0/16` aggregate, subnetted by VLAN ID for memorability.
 
| VLAN | Purpose | CIDR | Gateway | DHCP range | Reserved |
|---|---|---|---|---|---|
| 10 | Corporate | `10.10.10.0/24` | `10.10.10.1` | `.100–.200` | `.10` (DC), `.20` (workstation) |
| 20 | Development | `10.10.20.0/24` | `10.10.20.1` | `.100–.200` | `.10` (Win11 dev), `.20` (Ubuntu dev) |
| 66 | Attacker DMZ | `10.10.66.0/24` | `10.10.66.1` | n/a (static) | `.10` (Kali) |
| 99 | SOC Management | `10.10.99.0/24` | `10.10.99.1` | n/a (static) | `.10` (Wazuh), `.20` (Splunk) |
| 100 | OpenVPN clients | `10.10.100.0/24` | (pfSense) | `.2–.50` | — |
 
**DNS:** pfSense forwards to the AD Domain Controller (`10.10.10.10`) for the internal `.lab` zone and falls back to `1.1.1.1` for external resolution.
 
**NTP:** pfSense acts as the canonical time source for the lab; all VMs sync to their gateway.
 
---
 
## 6. Hardware & Resource Budget
 
### Host platform
 
- CPU: Intel Core i7-14700KF (20 cores, 28 threads)
- RAM: 32 GB DDR5
- Storage: NVMe (~250 GB allocated to VM disks)
- GPU: not used (lab is headless except for desktop sessions)
### RAM allocation per VM
 
| VM | RAM | Notes |
|---|---|---|
| pfSense | 2 GB | Including Suricata package overhead with ET Open ruleset |
| Windows Server 2022 (DC) | 4 GB | |
| Windows 11 Pro (Corp) | 4 GB | |
| Windows 11 Pro (Dev) | 4 GB | |
| Ubuntu Desktop 24.04 (Dev) | 3 GB | |
| Ubuntu Server (Wazuh) | 6 GB | Manager + Indexer + Dashboard on a single VM. Tight; 8 GB is the documented recommendation. |
| Ubuntu Server (Splunk) | 4 GB | Sufficient for personal lab event volumes. |
| Kali Linux | 4 GB | |
| **Total** | **31 GB** | |
 
Host operating system overhead: ~4 GB.
Combined demand if all VMs are powered on: ~35 GB → over budget by ~3 GB.
 
### Operating strategy
 
VMs are grouped by concurrent need:
 
- **Always on:** pfSense, Wazuh server, Splunk server, DC. **Total: 16 GB**
- **On during corporate scenarios:** Windows 11 Corp, Windows 11 Dev or Ubuntu Dev (one of the dev hosts). **Total: +7–8 GB**
- **On during attack scenarios:** Kali. **Total: +4 GB**
- **On during VPN/cross-VLAN testing:** Both dev VMs simultaneously.
The lab is designed so that no single attack scenario requires all eight VMs running concurrently. Upgrading to 64 GB is the medium-term solution if multi-scenario chaining becomes routine.
 
### Disk allocation (sparse-provisioned)
 
| VM | Allocated | Typical use |
|---|---|---|
| pfSense | 8 GB | 1–2 GB used |
| Windows Server 2022 | 60 GB | 25–30 GB |
| Windows 11 Pro × 2 | 50 GB each | 20–25 GB |
| Ubuntu Desktop | 25 GB | 10–12 GB |
| Ubuntu Server (Wazuh) | 40 GB | Grows with indices |
| Ubuntu Server (Splunk) | 40 GB | Grows with indices |
| Kali | 30 GB | 12–15 GB |
 
---
 
## 7. VLAN Design Summary
 
| VLAN | Tag | Purpose | Hosts |
|---|---|---|---|
| 10 | Corporate | Domain-joined users and identity services | DC + 1 workstation |
| 20 | Development | Isolated dev environment, accessible only via VPN/RDP | Win + Linux workstations |
| 66 | Attacker DMZ | Simulated external threat actor behind NAT | Kali |
| 99 | SOC Management | Out-of-band telemetry collection and analytics | Wazuh + Splunk |
 
The reasoning behind each VLAN is captured in the architecture decisions above.
 
---
 
## 8. Firewall Policy Principles
 
Detailed firewall rules are documented in [`phase2-network-backbone.md`](phase2-network-backbone.md). At the design level the policy is:
 
- **Default deny** between VLANs. All allowed flows are explicit.
- **Corporate → Development** allowed only on TCP 3389 (RDP) and UDP 1194 (OpenVPN).
- **Development → Corporate** denied by default. AD join requires explicit allowance for ports 53, 88, 389, 445, 464 against the DC only.
- **Any → SOC** allowed only on TCP 1514 (Wazuh agent) and TCP 8088 (Splunk HEC). Inbound RDP, SSH, SMB toward VLAN 99 are denied.
- **SOC → anywhere** denied. The SOC observes; it does not connect.
- **Attacker → anywhere** denied by default. Each attack scenario opens minimal scenario-specific rules.
- **VLAN 99 → Internet** allowed (Wazuh CTI feeds, Suricata rule updates, Splunk app downloads).
- **All denies are logged** and forwarded to Splunk for review.
---
 
## 9. Acceptance Criteria per Phase
 
Each phase has a measurable definition of done. A phase is not considered complete until all criteria are met.
 
| Phase | Definition of done |
|---|---|
| 1 — VirtualBox Foundation | Four Internal Networks exist; all required ISOs downloaded and checksummed; base VM templates created with snapshots. |
| 2 — Network Backbone | pfSense routes between all four VLANs; default-deny inter-VLAN policy in place; NAT outbound works; isolation verified from a live Ubuntu booted into each VLAN. |
| 3 — SOC Stack | Wazuh manager reachable from any agent; Splunk HEC accepts events with `useACK`; both platforms ingest test events. |
| 4 — Corporate Environment | AD domain operational; both Windows endpoints joined; Sysmon installed with the Olaf Hartong baseline; Wazuh agents reporting. |
| 5 — Development Environment | Both dev workstations on VLAN 20; agents reporting; no inbound paths from VLAN 20 except via VPN/RDP. |
| 6 — Attack Infrastructure | Kali on VLAN 66; baseline toolkit verified; outbound to corporate denied by default. |
| 7 — Remote Access (OpenVPN) | OpenVPN server functional; certificate-based authentication; corporate user can reach dev workstations; non-corp source cannot. |
| 8 — Detection Engineering | 15 rules deployed; each rule has a matching atomic test that fires it; each rule has a documented L1 runbook. |
| 9 — SOC Operations | Four end-to-end attack scenarios executed; each has a complete incident report with timeline, IOCs, MITRE mapping, and recommendations. |
| 10 — Portfolio Capstone | README polished; all phase docs cross-linked; LinkedIn posts published; demo material recorded. |
 
---
 
 
## Next phase
 
[Phase 1 — VirtualBox Foundation](phase1-virtualbox-foundation.md): provision the host hypervisor, define the four Internal Networks in VirtualBox, download installation media with verified checksums, and create base VM templates with snapshots for fast rebuild.
