# Scenario 1 — Full Kill Chain against the Dev VLAN
 
## Overview
 
The first attack scenario of the lab exercises the full kill chain against the development environment (VLAN 20): reconnaissance, credential brute force, initial access, host discovery, credential harvesting from local files, lateral movement to a Windows workstation, and two independent persistence layers. Ten distinct phases were executed end-to-end from the Adversary DMZ (Kali on VLAN 66), each phase mapped to a MITRE ATT&CK technique and each phase's telemetry captured in the SIEM.
 
The scenario is deliberately end-to-end. Rather than isolating a single technique, the intent is to demonstrate that the SOC pipeline responds coherently to a **realistic intrusion chain** — one where the attacker moves through the environment in stages, each stage building on the previous. The narrative parallels real incidents: a foothold established via credential attack becomes credential harvesting becomes lateral movement becomes persistence. Every stage produces telemetry, and the SIEM's job is to make each stage visible to the analyst investigating the incident.
 
The scenario was executed against pre-seeded targets: a Linux dev workstation (ws-dev-02) with synthetic credentials placed in developer-typical locations, and a Windows dev workstation (WS-DEV-01) with RDP enabled. This seeding is documented explicitly as environment preparation, not concealed as "found artefacts" — the intent is to model realistic developer workstations, not to fake attacker discoveries.
 
The scenario also surfaces an operational reality that dominates real SOCs: **alert fatigue**. The 10 phases generated 40,034 alerts, of which 87% were duplicates of a single detection rule triggered by the reconnaissance scan. This ratio, and the response to it, is documented at the end of this document as the primary lesson learned from executing the scenario at production-realistic scale.
 
---


## Kill Chain Phases
 
| Phase | Tactic | Actor | Target | MITRE Tecnic |
| ----- | ------ | --------- | ------------- | ------------- | 
| 1 | Reconnaissance | Kali | Subnet 10.10.20.0/24 | T1046 | 
| 2 | Brute Froce SSH |  Kali | ws-dev-02 (10.10.20.20) | T1110.001 | 
| 3 | Initial Access |  Kali | ws-dev-02 | T1078 | SSH |
| 4 | Discovery post-compromise | Kali (via ws-dev-02) | ws-dev-02 | T1082, T1087, T1057 |
| 5 | Credential Access | Kali (via ws-dev-02) | ws-dev-02 | T1552.001, T1552.003 | 
| 6 | Lateral Movement | ws-dev-02 (pivot) | WS-DEV-01 (10.10.20.10) | T1021.001 (RDP) |
| 7 | Persistence Linux | Kali (via ws-dev-02) | ws-dev-02 | T1053.003 (Cron) |
| 8 | Persistence SSH | Kali (via ws-dev-02) | ws-dev-02 | T1098.004 (SSH Authorized Keys) |


---
 
## MITRE ATT&CK coverage
 
| Phase | Tactic | Technique | Sub-technique |
| ----- | ------ | --------- | ------------- |
| 1 | Reconnaissance | T1046 — Network Service Discovery | — |
| 2 | Credential Access | T1110 — Brute Force | T1110.001 — Password Guessing |
| 3 | Initial Access | T1078 — Valid Accounts | — |
| 4 | Discovery | T1082 — System Information Discovery | — |
| 4 | Discovery | T1087 — Account Discovery | — |
| 4 | Discovery | T1057 — Process Discovery | — |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.001 — Credentials In Files |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.003 — Bash History |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.004 — Private Keys |
| 6 | Lateral Movement | T1021 — Remote Services | T1021.001 — Remote Desktop Protocol |
| 6 | Command and Control | T1572 — Protocol Tunneling | — |
| 7 | Persistence | T1053 — Scheduled Task/Job | T1053.003 — Cron |
| 8 | Persistence | T1098 — Account Manipulation | T1098.004 — SSH Authorized Keys |





