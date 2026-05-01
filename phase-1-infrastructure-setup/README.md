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
- **Adapter 2:** Internal Network — `Internal`

## Screenshots
### 1. Virtual Machines Overview and VMs Configuration
![4vms](screenshots/4vms.png) 
All 4 virtual machines created in VirtualBox: Kali Attack, Wazuh, Splunk and Windows Target.

#### Kali Attack — VM Configuration
![Descripcion](screenshots/01-vms-overview.png)
- OS: Ubuntu 64-bit (Kali Linux)
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

 #### Windows Target — VM Configuration
![Descripcion](screenshots/01-vms-overview.png)
- OS: Windows 11 64-bit
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Wazuh — VM Configuration
![Descripcion](screenshots/01-vms-overview.png)
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Splunk — VM Configuration
![Descripcion](screenshots/01-vms-overview.png)
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

### 2. Network — Configuration Adapter 1 NAT
![Descripcion](screenshots/01-vms-overview.png)
Adapter 1 is set to NAT mode to provide internet access inside each VM. 
This is necessary to be able to update the system, install packages and 
download tools throughout the project. Without this adapter, the VMs would 
be completely isolated with no ability to reach external repositories or 
download software.

### 3. Network — Configuration Adapter 2 Internal Network
![Descripcion](screenshots/01-vms-overview.png)
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
![Descripcion](screenshots/01-vms-overview.png)
To assign a static IP in Linux, the network configuration file must be edited 
manually via terminal. The file /etc/network/interfaces is modified to set 
a fixed IP address, subnet mask and interface for the Internal Network adapter.
![Descripcion](screenshots/01-vms-overview.png)
Once the configuration is applied and the network service restarted, the 
command ip a is used to verify that the static IP has been correctly assigned 
to the interface.


#### Target Windows
![Descripcion](screenshots/01-vms-overview.png)
To assign a static IP in Windows 11, the network adapter settings are 
accessed through Control Panel. The Internal Network adapter is configured 
with a fixed IP address and subnet mask under IPv4 properties.
![Descripcion](screenshots/01-vms-overview.png)
The command ipconfig in CMD confirms the static IP has been successfully 
assigned to the Internal Network adapter.


#### Wazuh
![Descripcion](screenshots/01-vms-overview.png)
To assign a static IP in Ubuntu Desktop, the network settings are accessed 
through the GUI. The Internal Network adapter is set to Manual mode with a 
fixed IP address and subnet mask.
![Descripcion](screenshots/01-vms-overview.png)
The command ip a confirms the static IP has been successfully applied 
to the enp0s8 interface.


#### Splunk
![Descripcion](screenshots/01-vms-overview.png)
To assign a static IP in Ubuntu Desktop, the network settings are accessed 
through the GUI. The Internal Network adapter is set to Manual mode with a 
fixed IP address and subnet mask.
![Descripcion](screenshots/01-vms-overview.png)
The command ip a confirms the static IP has been successfully applied 
to the enp0s8 interface.

### 5. Connectivity Verification
To verify that all static IPs have been correctly assigned and that all VMs 
can communicate through the Internal Network, ping tests are performed from 
the Kali Attack machine to each of the other VMs. This confirms that the 
network configuration is working as expected and that the lab environment 
is ready for the next phases. A successful ping response from each machine 
demonstrates full connectivity across the 192.168.10.0/24 subnet.

#### Linux > Windows Target
![Descripcion](screenshots/01-vms-overview.png)
Successful ping to 192.168.10.20 confirming connectivity with the Windows Target machine.

#### Linux > Wazuh
![Descripcion](screenshots/01-vms-overview.png)
Successful ping to 192.168.10.30 confirming connectivity with the Wazuh Manager.

#### Linux > Splunk
<img width="508" height="186" alt="image" src="https://github.com/user-attachments/assets/886602c6-91c2-4ea1-a9c6-1c82fec6fb42" />
Successful ping to 192.168.10.40 confirming connectivity with the Splunk SIEM.

