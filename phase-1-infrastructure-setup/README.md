# Phase 1 — Infrastructure Setup

## Objective
The goal of this phase is to deploy and configure the four virtual machines 
that form the foundation of the SOC lab environment. Each VM is assigned a 
specific role within the infrastructure: attacker, target, SIEM backend and 
SIEM frontend. All machines are connected through a shared Internal Network 
in VirtualBox, with static IPs assigned to each one to ensure stable 
connectivity across reboots. This phase establishes the base infrastructure 
on which all subsequent phases — Wazuh, Splunk, Snort and detection rules — 
will be built.

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

#### Kali Attack — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/47a8ea34-fbfe-458a-b0d8-4ef27f5ef2cd" />
- OS: Ubuntu 64-bit (Kali Linux)
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

 #### Windows Target — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/5cf1fbe7-f026-48bf-be9a-8ec5fc9ecaac" />
- OS: Windows 11 64-bit
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Wazuh — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/579afdf2-2182-430b-9c68-ab002cf7bc81" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Splunk — VM Configuration
<img width="450" height="500" alt="image" src="https://github.com/user-attachments/assets/e03df726-8466-401c-8e98-bfd7b1a292b8" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

### 2. Network — Configuration Adapter 1 NAT
<img width="400" height="190" alt="image" src="https://github.com/user-attachments/assets/51924df4-bdc4-415b-a46e-f6ce406cdd69" />
Adapter 1 is set to NAT mode to provide internet access inside each VM. 
This is necessary to be able to update the system, install packages and 
download tools throughout the project. Without this adapter, the VMs would 
be completely isolated with no ability to reach external repositories or 
download software.

### 3. Network — Configuration Adapter 2 Internal Network
<img width="400" height="190" alt="image" src="https://github.com/user-attachments/assets/dfb92db1-00f1-4c75-80b9-8022c77c4944" />
Adapter 2 is set to Internal Network mode to enable direct communication 
between all VMs within the lab environment. This creates an isolated internal 
network where the attack machine, target, Wazuh and Splunk can reach each 
other without going through the host machine or the internet. Combined with 
the static IP configuration, this ensures that connectivity between VMs is 
preserved across reboots, so Wazuh Agent always knows where the Manager is 
and Splunk always receives data from the expected sources.

### 4. Assigned Static IP
Static IPs are manually assigned to each VM on the Internal Network adapter 
to ensure stable communication across reboots. Without static IPs, DHCP could 
assign a different IP each time a VM restarts, breaking the connection between 
Wazuh Agent and Wazuh Manager, or between Wazuh and Splunk. Each VM receives 
a fixed IP on the 192.168.10.0/24 subnet. Below is the configuration applied 
to each machine followed by a verification screenshot confirming the IP was 
correctly assigned.

#### Kali Attack
<img width="500" height="283" alt="image" src="https://github.com/user-attachments/assets/38781ffd-8aca-4462-8281-188eb7d466b3" />
<img width="500" height="283" alt="image" src="https://github.com/user-attachments/assets/776a0a06-1f59-4c5d-906d-cec26e482dd6" />


#### Target Windows
#### Wazuh
<img width="771" height="481" alt="image" src="https://github.com/user-attachments/assets/f4884e31-5b8b-46a5-858d-a9f2ab9c6213" />
<img width="771" height="481" alt="image" src="https://github.com/user-attachments/assets/f780330c-3759-4571-84c8-6b8594a644f0" />


#### Splunk
<img width="1285" height="801" alt="image" src="https://github.com/user-attachments/assets/c2a89176-c0f2-433d-8642-78838fbabe31" />
<img width="1038" height="573" alt="image" src="https://github.com/user-attachments/assets/27b48f75-ecf8-4127-be19-5b4fde5a5742" />


