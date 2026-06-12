# SOC-HomeLab
 
> A fully integrated Security Operations Center home lab built from scratch with VLAN segmentation, covering the complete threat detection and response lifecycle. Combines Blue Team detection engineering, Red Team attack simulation, and incident response. Oriented toward SOC Analyst, Detection Engineer, Threat Hunter roles and Pentester.
 
---
 
## Overview
 
This lab implements a production-grade SOC architecture across eight virtual machines distributed in four isolated VLANs, routed and firewalled through pfSense. The full pipeline includes perimeter and inter-VLAN intrusion detection (Suricata on pfSense), endpoint monitoring (Wazuh), centralized log ingestion (Splunk via HEC), enhanced Windows telemetry (Sysmon), and an Active Directory environment for realistic attack simulation. Network segmentation enforces an out-of-band SOC management plane, a dedicated attacker DMZ, and controlled cross-VLAN access between corporate and development networks.
 
Beyond infrastructure, the project covers all stages of the SOC analyst workflow — from rule writing and attack execution to forensic investigation, hunt operations, automated response, and threat intelligence enrichment.
 
---
 
## Architecture

```mermaid
Para hacer que el recuadro de la VLAN 99 (SOC Management) destaque visualmente y luzca mucho más profesional, podemos aplicar técnicas avanzadas de diseño en Mermaid:

Jerarquía Visual Inversa: Le daremos un fondo oscuro profundo diferenciado (#0f1115) con un borde naranja neón brillante (#ff9100) de doble grosor (stroke-width:2.5px).

Iconografía de Bloque: Utilizaremos un título formateado con carácteres limpios y una separación interna más elegante.

Efecto de Enfoque: Al oscurecer sutilmente el resto de las VLANs de la red interna, la zona del SOC se convertirá automáticamente en el centro de atención visual del diagrama para cualquiera que visite tu GitHub.

Aquí tienes el código de Mermaid optimizado y listo para tu README.md:

Fragmento de código
graph TD
    %% ==========================================
    %% GLOBAL CLASS DEFINITIONS & GRAPH THEME
    %% ==========================================
    classDef perimeter fill:#b30000,stroke:#222,stroke-width:1.5px,color:#fff;
    classDef ingest fill:#007acc,stroke:#005a9e,stroke-width:1.5px,color:#fff;
    classDef siem fill:#f57c00,stroke:#e65100,stroke-width:1.5px,color:#fff;
    classDef winNode fill:#2b579a,stroke:#1e3a8a,stroke-width:1.2px,color:#fff;
    classDef nixNode fill:#2ca02c,stroke:#1b5e20,stroke-width:1.2px,color:#fff;
    classDef redNode fill:#4d4d4d,stroke:#1a1a1a,stroke-width:1.5px,color:#fff;
    classDef transit fill:#22252a,stroke:#7f8c8d,stroke-width:1px,color:#fff;

    %% ==========================================
    %% EDGE & BOUNDARY CONTROL
    %% ==========================================
    WAN_IN((Internet / External)) --> FW_CORE

    subgraph SECURITY_EDGE [Perimeter Edge Security]
        FW_CORE[pfSense CE<br>Edge Router / VPN Gateway]:::perimeter
        IDS_CORE["Suricata NIDS Engine<br>Inline Inspection Tier<br>Virtual Interface IP: 10.10.X.1"]:::perimeter
        FW_CORE --> IDS_CORE
    end

    %% ==========================================
    %% PRIVILEGED & ENDPOINT ZONE INFRASTRUCTURE
    %% ==========================================
    subgraph VLAN_10 [VLAN 10: Corporate Domain]
        DC_2022["Windows Server 2022<br>10.10.10.10<br>Active Directory DC / DNS<br>Sysmon + Wazuh Agent"]:::winNode
        CL_CORP["Windows 11 Pro<br>10.10.10.20<br>Corporate Workstation<br>Sysmon + Wazuh Agent"]:::winNode
    end

    subgraph VLAN_20 [VLAN 20: Software Development]
        CL_DEV_W["Windows 11 Pro<br>10.10.20.10<br>Dev Workstation<br>Sysmon + Agent"]:::winNode
        CL_DEV_U["Ubuntu Desktop 24.04<br>10.10.20.20<br>Dev Engineering Host<br>Auditd + Agent"]:::nixNode
    end

    %% ==========================================
    %% HIGH-VISIBILITY SOC ZONE
    %% ==========================================
    subgraph VLAN_99 [🛡️ MULTI-TENANT SOC PLATFORM — VLAN 99]
        SOC_WAZUH["Wazuh SIEM Manager<br>10.10.99.10<br>XDR Core / Indexer Node<br>TCP Ports: 1514, 55000"]:::ingest
        SOC_SPLUNK["Splunk Enterprise SIEM<br>10.10.99.20<br>Central Analytics Engine<br>TCP Ports: 8000, 8088"]:::siem
        SOC_WAZUH -->|HEC Event Stream| SOC_SPLUNK
    end

    %% ==========================================
    %% UNTRUSTED DMZ ZONE
    %% ==========================================
    subgraph VLAN_66 [VLAN 66: Adversary Emulation DMZ]
        ATK_KALI["Kali Linux Platform<br>10.10.66.10<br>Offensive Operations Host<br>Gateway Local: 10.10.66.1"]:::redNode
    end

    %% ==========================================
    %% TELEMETRY BUS
    %% ==========================================
    LOG_BUS["Unified Telemetry Pipeline<br>Unidirectional Logging Bus"]:::transit

    %% Inter-VLAN Routing Vectors
    IDS_CORE -->|Subnet Transit| VLAN_10
    IDS_CORE -->|Subnet Transit| VLAN_20
    IDS_CORE -->|Subnet Transit| VLAN_99

    %% Adversary Engagement Vector
    ATK_KALI ==>|Targeted Adversary Traversal| IDS_CORE
    
    %% Telemetry Collection Mappings
    DC_2022 -.-> LOG_BUS
    CL_CORP -.-> LOG_BUS
    CL_DEV_W -.-> LOG_BUS
    CL_DEV_U -.-> LOG_BUS
    LOG_BUS ==>|Structured JSON Ingestion| SOC_WAZUH

    %% ==========================================
    %% ADVANCED CONTAINERS PRESETS (PREMIUM DARK)
    %% ==========================================
    style VLAN_10 fill:#111418,stroke:#1f77b4,stroke-width:1.2px,color:#999;
    style VLAN_20 fill:#111418,stroke:#2ca02c,stroke-width:1.2px,color:#999;
    style VLAN_66 fill:#1a1313,stroke:#d62728,stroke-width:1.2px,color:#fff;
    style SECURITY_EDGE fill:#191111,stroke:#b30000,stroke-width:1.5px,color:#fff;
    
    %% Premium Cyber-SOC Neon Styling
    style VLAN_99 fill:#0f1115,stroke:#ff9100,stroke-width:2.5px,color:#fff;
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
