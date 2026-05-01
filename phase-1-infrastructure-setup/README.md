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
