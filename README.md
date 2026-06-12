# SOC-HomeLab
 
> A fully integrated Security Operations Center home lab built from scratch with VLAN segmentation, covering the complete threat detection and response lifecycle. Combines Blue Team detection engineering, Red Team attack simulation, and incident response. Oriented toward SOC Analyst, Detection Engineer, Threat Hunter roles and Pentester.
 
---
 
## Overview
 
This lab implements a production-grade SOC architecture across eight virtual machines distributed in four isolated VLANs, routed and firewalled through pfSense. The full pipeline includes perimeter and inter-VLAN intrusion detection (Suricata on pfSense), endpoint monitoring (Wazuh), centralized log ingestion (Splunk via HEC), enhanced Windows telemetry (Sysmon), and an Active Directory environment for realistic attack simulation. Network segmentation enforces an out-of-band SOC management plane, a dedicated attacker DMZ, and controlled cross-VLAN access between corporate and development networks.
 
Beyond infrastructure, the project covers all stages of the SOC analyst workflow — from rule writing and attack execution to forensic investigation, hunt operations, automated response, and threat intelligence enrichment.
 
---
 
## Architecture

```mermaid
graph TD
    %% Configuración de Orientación
    direction TB

    %% Estilos Globales y de Nodos
    classDef firewall fill:#b30000,stroke:#333,stroke-width:2px,color:#fff;
    classDef windows fill:#1f77b4,stroke:#333,stroke-width:1px,color:#fff;
    classDef linux fill:#2ca02c,stroke:#333,stroke-width:1px,color:#fff;
    classDef soc fill:#ff7f0e,stroke:#333,stroke-width:1px,color:#fff;
    classDef kali fill:#4d4d4d,stroke:#333,stroke-width:2px,color:#fff;
    classDef zone fill:#f9f9f9,stroke:#666,stroke-width:1px,stroke-dasharray: 5 5;

    %% Perímetro / WAN
    Internet((Internet)) <--> |WAN| pfSense

    %% Core de Red (Firewall Centralizado)
    pfSense[pfSense CE<br>10.10.X.1<br>Core Router / Firewall / IDS]:::firewall

    %% Segmentos de Red (Zonas Aisladas Lógicamente)
    
    subgraph V10 [VLAN 10: Corp Network]
        WinServer["Windows Server 2022<br>10.10.10.10<br>(AD DC / DNS)"]:::windows
        Win11Corp["Windows 11 Pro<br>10.10.10.20<br>(Workstation)"]:::windows
    end

    subgraph V20 [VLAN 20: Dev Network]
        Win11Dev["Windows 11 Pro<br>10.10.20.10<br>(Workstation)"]:::windows
        UbuntuDesk["Ubuntu Desktop 24.04<br>10.10.20.20<br>(Workstation)"]:::linux
    end

    subgraph V66 [VLAN 66: Attack Zone]
        Kali["Kali Linux<br>10.10.66.10<br>(Pentest Machine)"]:::kali
    end

    subgraph V99 [VLAN 99: SOC / Management]
        Wazuh["Wazuh Server<br>10.10.99.10<br>(Manager / Indexer)"]:::soc
        Splunk["Splunk SIEM<br>10.10.99.20<br>(Enterprise SIEM)"]:::soc
    end

    %% Enlaces Troncales Únicos al pfSense (Representa el etiquetado 802.1Q)
    pfSense <==> |Tráfico Inter-VLAN| V10
    pfSense <==> |Tráfico Inter-VLAN| V20
    pfSense <==> |Tráfico Inter-VLAN| V66
    pfSense <==> |Tráfico Inter-VLAN| V99

    %% Estilos de los contenedores para documentación limpia
    style V10 fill:#f5f9fc,stroke:#1f77b4,stroke-width:1px;
    style V20 fill:#f5fcf5,stroke:#2ca02c,stroke-width:1px;
    style V66 fill:#fafafa,stroke:#4d4d4d,stroke-width:1px;
    style V99 fill:#fffbf5,stroke:#ff7f0e,stroke-width:1px;
```
---
 
## Lab Components
 
| VM | IP | OS | Role |
|----|----|----|------|
| pfSense | 10.10.X.1 (per VLAN) | pfSense CE (latest) | Router + Firewall + Suricata IDS + OpenVPN |
| Windows Server 2022 | 10.10.10.10 | Windows Server 2022 (Eval) | Active Directory DC + DNS + Sysmon + Wazuh Agent |
| Windows 11 Pro (Corp) | 10.10.10.20 | Windows 11 Pro | Corporate workstation + Sysmon + Wazuh Agent |
| Windows 11 Pro (Dev) | 10.10.20.10 | Windows 11 Pro | Development workstation + Sysmon + Wazuh Agent |
| Ubuntu Desktop | 10.10.20.20 | Ubuntu Desktop 24.04 | Development workstation + Auditd + Wazuh Agent |
| Ubuntu Wazuh | 10.10.99.10 | Ubuntu Server 24.04 | Wazuh Manager + Indexer + Dashboard |
| Ubuntu Splunk | 10.10.99.20 | Ubuntu Server 24.04 | Splunk Enterprise SIEM |
| Kali Linux | 10.10.66.10 | Kali Linux (latest) | Attack machine |
 
---
 
## Project Phases
 
| # | Phase | Description |
|---|-------|-------------|
| 0 | [Planning & Design](docs/phase0-planning-design.md) | Architecture decisions, threat model, IP plan, scope |
| 1 | [VirtualBox Foundation](docs/phase1-virtualbox-foundation.md) | Hypervisor setup, internal networks, base VM provisioning |
| 2 | [Network Backbone](docs/phase2-network-backbone.md) | pfSense install + VLAN segmentation + firewall policy |
| 3 | [SOC Stack Deployment](docs/phase3-soc-stack.md) | Wazuh Manager + Splunk SIEM + HEC integration |
| 4 | [Corporate Environment](docs/phase4-corporate-environment.md) | Active Directory + Windows endpoints + Sysmon |
| 5 | [Software Development Environment](docs/phase5-development-environment.md) | Windows + Linux dev workstations in isolated VLAN |
| 6 | [Attack Infrastructure](docs/phase6-attack-infrastructure.md) | Kali Linux setup + offensive toolkit baseline |
| 7 | [Remote Access (OpenVPN)](docs/phase7-remote-access.md) | OpenVPN server on pfSense + cross-VLAN access policy |
| 8 | [Detection Engineering Baseline](docs/phase8-detection-engineering.md) | 15+ custom detection rules mapped to MITRE ATT&CK |
| 9 | [SOC Operations & Incident Reporting](docs/phase9-soc-operations.md) | End-to-end attack scenarios + incident response reports |
| 10 | [Portfolio Capstone](docs/phase10-portfolio-capstone.md) | Final writeup, demo materials, and portfolio polish |
 
---
 
## Tools Used
 
| Category | Tools |
|----------|-------|
| **Network** | pfSense (router + firewall + VPN) |
| **SIEM** | Splunk Enterprise |
| **EDR** | Wazuh |
| **IDS** | Suricata |
| **Windows Telemetry** | Sysmon (Olaf Hartong config) |
| **Identity** | Active Directory (Windows Server 2022) |
| **Offensive** | Kali Linux, Nmap, Hydra, Metasploit, Burp Suite, John the Ripper, Hashcat |
| **Forensics** | Volatility, Wireshark |

---
 
## Repository Structure
 
```
SOC-HomeLab/
├── docs/                       Phase-by-phase build documentation
├── detection-engineering/      Detection rules with full methodology
├── rules/                      Ready-to-deploy rule code (Wazuh XML, Splunk SPL)
└── screenshots/                Visual evidence per phase
```
 
---
 
## About This Project
 
Built as a personal cyber range to develop and demonstrate the full skill set required for SOC Analyst and Detection Engineer roles. The lab is continuously evolving — new rules, attack scenarios, and automation playbooks are added as part of ongoing learning. Network segmentation, out-of-band SOC management, and a dedicated attacker DMZ make this lab a realistic environment for both detection engineering and incident response practice. All documentation is written in English to align with international industry standards.
