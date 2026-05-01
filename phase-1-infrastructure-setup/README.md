# Phase 1 — Infrastructure Setup

## Objective
Deploy and configure all virtual machines in VirtualBox and establish 
internal network connectivity between them.

## Virtual Machines

| VM | OS | Role | RAM | CPUs | IP |
|---|---|---|---|---|---|
| Kali Attack | Kali Linux | Attacker | 4GB | 2 | 192.168.10.10 |
| Windows Target | Windows 11 Pro | Victim + Wazuh Agent | 4GB | 2 | 192.168.10.20 |
| Wazuh HomeLab | Ubuntu Desktop | Wazuh Manager | 6GB | 4 | 192.168.10.30 |
| Splunk HomeLab | Ubuntu Desktop | Splunk SIEM | 6GB | 4 | 192.168.10.40 |

## Network Configuration
- **Adapter 1:** NAT (internet access)
- **Adapter 2:** Internal Network — `Interal`

## Screenshots
### 1. Virtual Machines Overview and VMs Configuration
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/ac84fd8b-41ce-4c13-b8bc-7f6553e7c3a0" />
All 4 virtual machines created in VirtualBox: Kali Attack, Wazuh, Splunk and Windows Target.

## Kali Attack — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/47a8ea34-fbfe-458a-b0d8-4ef27f5ef2cd" />
- OS: Ubuntu 64-bit (Kali Linux)
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

## Windows Target — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/5cf1fbe7-f026-48bf-be9a-8ec5fc9ecaac" />
- OS: Windows 11 64-bit
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

## Wazuh — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/579afdf2-2182-430b-9c68-ab002cf7bc81" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

## Splunk — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/e03df726-8466-401c-8e98-bfd7b1a292b8" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

