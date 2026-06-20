# SOC HomeLab — From Infrastructure to Incident Analysis

> A personal Security Operations Center built from scratch in a fully segmented virtualized environment to build SOC L1 skill. The lab replicates a small enterprise architecture with corporate, development, attacker and SOC management VLANs, and is operated as a real SOC — including detection engineering, attack simulation, and incident response. 

### Architectural Evolution

This lab represents a major iteration over my previous HomeLab environment. Through ongoing research into enterprise network security, I identified that a flat network architecture (where all assets sit in a single subnet) constitutes a critical vulnerability and fails to reflect real-world corporate environments. 

To address this gap and implement a true "Defense in Depth" strategy, this project leverages strict VLAN segmentation. By isolating corporate users, software development assets, and SOC management tools into distinct trust zones, the infrastructure restricts lateral movement. If an attacker compromises a single host or VLAN, network access policies prevent them from easily escalating privileges or gaining control over the entire organization.

## Why this project?

I want to demonstrate that I understand the full security operations cycle: how a realistic environment is built, how an attacker reasons, how an intrusion is detected with real telemetry, and how an incident is documented professionally.

A course teaches you to use tools. This lab forces me to use them in a system I built myself.

## What I'm going to do

1. **Build** a segmented corporate network with an Active Directory domain, Windows and Linux endpoints, and a pfSense + Suricata perimeter.
2. **Deploy** a full SIEM/EDR stack (Wazuh all-in-one) with telemetry from every host.
3. **Attack** from a DMZ with Kali, executing techniques mapped to MITRE ATT&CK.
4. **Detect** those techniques in Wazuh, and write custom rules where the baseline detection falls short.
5. **Report** every incident using the format a real L1 analyst would use.

## Architecture

![SOC HomeLab Architecture](00-architecture/architecture.svg)

| Segment  | CIDR           | Purpose                                    |
| -------- | -------------- | ------------------------------------------ |
| VLAN 10  | 10.10.10.0/24  | Corporate domain (AD DC + workstation)     |
| VLAN 20  | 10.10.20.0/24  | Software Development (Windows + Ubuntu)    |
| VLAN 66  | 10.10.66.0/24  | Attacker DMZ (Kali Linux)                  |
| VLAN 99  | 10.10.99.0/24  | SOC Management (Wazuh all-in-one)          |

Full reasoning behind each choice in [`00-architecture/design-decisions.md`](00-architecture/design-decisions.md).

## Roadmap

| Phase                                |  Content                                            |
| ------------------------------------ |-------------------------------------------------- |
| 0 — Design and prerequisites         | Diagram, decisions, checklist                      |
| 1 — Base infrastructure              | pfSense, AD, Windows and Linux endpoints           |
| 2 — SOC stack                        | Wazuh Manager + Indexer + Dashboard, Suricata      |
| 3 — Adversary                        | Kali Linux + offensive toolkit                     |
| 4 — Attack scenarios and detection   | Execution, detection, tuning, incident report      |

## Planned attack scenarios

Each scenario includes: attack execution, observed detection, custom detection rule (where the native one isn't enough), and an L1-format incident report.

| #  | Technique                              | MITRE ATT&CK |
| -- | -------------------------------------- | ------------ |
| 01 | Network reconnaissance                 | T1046        | 
| 02 | Phishing with macro payload            | T1566.001    |
| 03 | Lateral movement via SMB / RDP         | T1021.002    |
| 04 | Credential dumping (LSASS)             | T1003.001    | 
| 05 | Exfiltration over alternative channel  | T1048        |

As I continue my cybersecurity journey, new attack methods and techniques will be added to this homelab.

## Tech stack

- **Perimeter:** pfSense CE · Suricata IDS · OpenVPN
- **Identity:** Active Directory (Windows Server 2022)
- **Endpoints:** Windows 11 Pro · Ubuntu Desktop 24.04
- **SIEM / EDR:** Wazuh Manager + Indexer + Dashboard (all-in-one)
- **Endpoint telemetry:** Sysmon (Windows) · Auditd (Linux)
- **Adversary:** Kali Linux + toolkit 
- **Virtualization:** Oracle VirtualBox

## Capabilities Demonstrated
 
- **Network architecture** — VLAN design, firewall policy, inter-segment routing, perimeter detection
- **SOC operations** — out-of-band management, telemetry pipelines, log forwarding, HEC integration
- **Detection engineering** — custom rule development, MITRE ATT&CK mapping, atomic testing, threshold calibration
- **Blue Team operations** — alert triage, incident investigation, forensic timeline reconstruction
- **Purple Team workflow** — offensive simulation feeding detection improvement
- **Documentation** — professional technical writing, decision rationale, reproducible deployment procedures

## Repository structure

```text
soc-homelab/
├── README.md
├── 00-architecture/        
├── 01-infrastructure/       
├── 02-soc-stack/            
├── 03-adversary/            
├── 04-attack-scenarios/    
├── 05-detection-rules/     
└── 06-incident-reports/     
```

## Project Phases

[`00-architecture`](00-architecture/)
[`01-infrastructure`](01-infrastructure/)
[`02-soc-stack`](02-soc-stack/)
[`03-adversary`](03-adversary/)
[`04-attack-scenarios`](04-attack-scenarios/)
[`05-detection-rules`](05-detection-rules/)
[`06-incident-reports`](06-incident-reports/)

## About This Project
 
Built as a personal cyber range to develop and demonstrate the full skill set required for SOC Analyst and Detection Engineer roles. The lab is continuously evolving — new rules, attack scenarios, and automation playbooks are added as part of ongoing learning. Network segmentation, out-of-band SOC management, and a dedicated attacker DMZ make this lab a realistic environment for both detection engineering and incident response practice. All documentation is written in English to align with international industry standards.
