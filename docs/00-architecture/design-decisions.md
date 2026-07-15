# Design decisions

This document captures the why behind the architecture. 

## 1. Why VLAN segmentation?

An attacker sitting in my DMZ shouldn't see raw traffic from my corporate workstations. Segmentation forces inter-segment traffic through pfSense, where I can:

- Apply granular firewall rules by direction and protocol.
- Inspect traffic with Suricata.
- Generate useful logs that flow into the SIEM.

Without segmentation I lose visibility of the lateral movement I specifically want to detect — and a flat lab doesn't let me practice realistic incident reports.

**Accepted trade-off:** higher initial configuration complexity.

## 2. Why Wazuh and not Splunk / pure ELK?

- **Wazuh is one of the most-used open-source SIEMs** in junior and mid-tier environments — I want to learn the tool I'm most likely to touch in a real job.
- **All-in-one** (Manager + Indexer + Dashboard) fits in a single VM, which lowers infrastructure cost.
- It combines **SIEM + EDR + FIM + vulnerability detection** in one package; covering this with pure ELK would force me to integrate several components by hand.
- Rules are written in readable XML, not a proprietary DSL like SPL. The learning curve is transferable.

Splunk would be more "professional" but the free license is heavily capped, and I don't want vendor lock-in on an educational project.

## 3. Why Sysmon + Auditd and not just the Wazuh agent?

The Wazuh agent collects native OS logs. That's not enough for a real SOC:

- On Windows, native events are verbose but limited — I don't see child processes with their command line, I don't see pipe creation, I don't see network connections per process.
- **Sysmon** fills exactly those gaps, and Sysmon events are well covered by public detection rules (SwiftOnSecurity baseline config, Olaf Hartong's ruleset, MITRE mapping).
- On Linux, **Auditd** gives me equivalent visibility over syscalls, executions, and modifications of sensitive files.

Without this telemetry layer, many MITRE techniques would be invisible to me. That would turn the lab into theater: I run the attack but fail to detect it because of missing logs, not lack of intelligence.

## 4. Why an Ubuntu host in the development VLAN?

Realism. A real corporate environment is heterogeneous. Limiting myself to Windows would leave me without practice on:

- Detection on Linux (Auditd, journald, SSH events).
- Cross-platform movement (Linux compromise → Windows pivot).
- Techniques specific to dev environments: exposed SSH keys, git repositories with secrets, containers as a vector.

## 5. Why MITRE ATT&CK as the scenario framework?

It's the language every professional SOC uses to describe techniques. Any incident report, detection rule, or threat hunt in production maps back to ATT&CK. If I build my scenarios around the framework, my documentation speaks the language an interviewer expects to hear from the first answer.

## 6. Why VirtualBox and not Proxmox / ESXi?

Pragmatic choice:

- It's free and runs on my current OS without reinstalling.
- Lower learning curve, which lets me spend time on SOC content rather than the hypervisor.
- For a personal 6-VM lab it's enough.

**Trade-offs explicitly accepted:**

- VirtualBox **doesn't support native 802.1Q VLAN tagging** the way Proxmox or ESXi do. The "VLANs" from the diagram are implemented in VirtualBox as separate **Internal Networks**. Functionally equivalent from pfSense's perspective (it remains the only router between them), but conceptually they're not real VLANs — they're hypervisor-level isolated networks.
- Lower performance than a Type-1 hypervisor.
- More limited networking (no LACP, no SR-IOV, no 802.1Q trunks).

If I scale the lab later, I'll migrate to Proxmox and document the transition as an extra chapter of the repo.

## 7. Why is the management VLAN (99) separated out-of-band?

The Wazuh console **shouldn't be reachable from the segments where the monitored workloads run**. If an attacker compromises a workstation and finds the SIEM on the same subnet, the next logical step is to try to compromise it — to wipe their tracks or blind the analyst.

An out-of-band VLAN with strict rules (telemetry only inbound from agents on TCP 1514/1515, management only from my admin machine over HTTPS) shrinks that attack surface. It's a basic principle of SOC architecture and I want the lab to reflect it from day one.

## 8. Why OpenVPN as the bridge between VLAN 10 and VLAN 20?

VLAN 10 (Corporate) and VLAN 20 (Software Development) are separate trust zones, but in a real company they're not isolated — corporate users (PMs, analysts, support) need to reach dev workstations to run RDP sessions, troubleshoot builds, or assist developers. Two ways to allow that crossing:

**Option A (rejected):** open a direct firewall rule allowing VLAN 10 → VLAN 20 on RDP / SSH. Simple, but:

- Any compromised endpoint on VLAN 10 has automatic line-of-sight into dev — too generous.
- No additional authentication layer beyond Windows credentials.
- Hard to audit *who* crossed *when*.

**Option B (chosen):** keep VLAN 10 → VLAN 20 denied by default at the firewall, and require an **OpenVPN tunnel** terminated on pfSense. Corporate users who need dev access install the OpenVPN client, authenticate, and only then get a route to VLAN 20.

Benefits:

- **Defense in depth.** Even if a VLAN 10 host is compromised, the attacker still needs valid VPN credentials to pivot.
- **Auditable.** pfSense logs every VPN connection (user, source IP, timestamp). Wazuh ingests those logs, creating a high-fidelity signal that's hard to bypass.
- **Realistic enterprise pattern.** Controlled crossings between trust zones via authenticated tunnels are standard in segmented enterprise networks.
- **Attack scenario opportunity.** "Compromised VPN credentials → unauthorized dev access" becomes a credible Phase 4 exercise — much more interesting than walking through an open firewall rule.

**Implementation overview:**

- OpenVPN server on pfSense, listening on the VLAN 10 gateway interface (`10.10.10.1:1194/UDP`).
- Tunnel subnet: `10.10.50.0/24` (logical overlay handled by pfSense + OpenVPN; not a VirtualBox Internal Network).
- Pushed route to clients: `10.10.20.0/24`.
- Firewall rules on pfSense restrict tunnel clients to specific services on VLAN 20 (RDP `3389`, SSH `22`). I configure these manually as the lab evolves rather than locking them in upfront.

## Updates

This section captures decisions taken after the initial end-to-end lab was complete.

### Phase 7: Extending to a complete SOC platform

Phases 1 to 6 delivered a working end-to-end detection engineering loop: segmented infrastructure, Wazuh SIEM with custom rules, an adversary environment, a full kill-chain attack scenario, ten MITRE-mapped detection rules, and an L1-level incident report. That closed the fundamentals loop for a junior SOC portfolio.

Phase 7 extends the lab with the tooling that turns a SIEM-only environment into a complete SOC platform:

- **Suricata** as network IDS deployed as a native pfSense package on VLANs 10, 20 and 66.
- **TheHive** as case management platform, running in Docker on a new dedicated VM (`soc-platform`, 10.10.99.20).
- **Cortex** as automated analysis engine alongside TheHive, providing IoC enrichment via analyzers.
- **MISP** as threat intelligence platform on the same Docker host, feeding IoC context back to Cortex.

**Why now and not from day one:**

- **Detection has to exist before response tooling makes sense.** A case management platform with no meaningful alerts turns into an empty backlog. TheHive without Wazuh producing quality signals would be shelfware. Deploying it now, on top of the ten custom rules from Phase 5, means every case that lands there has real content behind it.
- **A SOC analyst's job doesn't end at the SIEM dashboard.** In a production SOC, a relevant alert triggers case creation, gets enriched with WHOIS / VirusTotal / MISP context, and is worked through to closure. Documenting only the detection side leaves the L1 role portrait incomplete. Adding TheHive + Cortex + MISP reproduces the flow that a real L1 executes every day.
- **Network-based detection complements host-based detection.** Wazuh + Sysmon + Auditd cover the endpoint layer well. Suricata adds signature-based visibility over the traffic itself — port scans, exploitation attempts on the wire, C2 patterns — which the host agents can miss when the payload never lands on disk. Combining EDR/SIEM with NDR is a standard defence-in-depth pattern in real SOCs.

**As my knowledge keeps growing, more updates will come out.**

