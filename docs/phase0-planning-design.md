# Phase 0 — Planning & Design
 
> **Goal:** Define the architecture, scope, and design decisions for the lab before any infrastructure is built. This document is the spec that subsequent phases follow.
 
## Overview
 
This phase produces no infrastructure. It consolidates the threat model, IP plan, hardware budget, and the rationale behind every architectural choice. Anyone reading this document should understand *why* the lab is structured the way it is — not just *what* gets built.
 
---
 
## 1. Objectives
 
The lab supports four concrete outcomes:
 
1. **Build a production-grade SOC** in miniature, with realistic network segmentation, telemetry pipelines, and out-of-band management.
2. **Develop Blue Team skills** end-to-end: detection engineering, threat detection, alert triage, incident response, and forensic investigation.
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
 
**Rationale:** Models realistic enterprise segmentation. Enables non-trivial firewall policy and inter-VLAN lateral movement detection scenarios. Without segmentation, the problem space collapses to single-subnet packet inspection, which is not how modern SOCs operate. That was a mistake I made in my last HomeLab.
 
**Alternative considered:** Single flat network at `192.168.10.0/24`

**Why rejected:** Eliminates the most important detection problem space — lateral movement across trust zones.
 
### 4.2 pfSense as the routing and firewall plane
 
**Decision:** pfSense CE on a dedicated VM with five interfaces (WAN + four LAN, one per VLAN).
 
**Rationale:** Free, well-documented, supports L3 inter-VLAN routing, has a mature package ecosystem (Suricata, OpenVPN, pfBlockerNG), and matches real-world SMB deployment patterns.
 
### 4.3 Suricata IDS on pfSense, not as a separate VM
 
**Decision:** Suricata runs as a pfSense package in IDS (passive) mode, monitoring all LAN interfaces.
 
**Rationale:** Single VM, lower RAM cost (Suricata shares pfSense resources), sees all inter-VLAN traffic at the routing point. In a 32 GB RAM budget this is the only economical choice.
 
**Alternative considered:** Separate Suricata VM with a port mirror from pfSense LAN switch.

**Why rejected:** Adds 4 GB RAM cost and significant configuration complexity. 
 
### 4.4 Out-of-band SOC management plane
 
**Decision:** Wazuh and Splunk live in VLAN 99 with strict push-only policy. Agents on data-plane hosts establish outbound TCP connections to the manager; the SOC VLAN never initiates connections into corporate or development.
 
**Rationale:** An attacker who compromises a corporate workstation cannot pivot into the monitoring infrastructure. If the SOC VLAN were reachable, the attacker could disable agents, tamper with logs, or pivot through the dashboard.
 
**Real-world parallel:** Standard SOC architecture; aligns with NIST SP 800-92 §4.3 (log management infrastructure isolation).
 
### 4.5 Dedicated attacker DMZ with NAT-simulated external access
 
**Decision:** Kali in VLAN 66, behind pfSense NAT. All "external" attacks originate from this VLAN and traverse pfSense (and Suricata) to reach data-plane VLANs.
 
**Rationale:** Forces realistic perimeter detection paths — Suricata edge alerts, NAT-translated source IPs, unauthorized RDP attempts.
 
### 4.6 VirtualBox hypervisor
 
**Decision:** VirtualBox 7.2.2 on a Windows host. Each VLAN modeled as a VirtualBox Internal Network.
 
**Rationale:** Free, available, runs on the existing workstation OS without dual-boot.
 
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
 
## 6. VLAN Design Summary
 
| VLAN | Tag | Purpose | Hosts |
|---|---|---|---|
| 10 | Corporate | Domain-joined users and identity services | DC + 1 workstation |
| 20 | Development | Isolated dev environment, accessible only via VPN/RDP | Win + Linux workstations |
| 66 | Attacker DMZ | Simulated external threat actor behind NAT | Kali |
| 99 | SOC Management | Out-of-band telemetry collection and analytics | Wazuh + Splunk |
 
The reasoning behind each VLAN is captured in the architecture decisions above.
 
---
 
## 7. Firewall Policy Principles
 
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
 
## Next phase
 
[Phase 1 — VirtualBox Foundation](phase1-virtualbox-foundation.md)
