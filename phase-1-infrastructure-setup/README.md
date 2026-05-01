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
<img width="449" height="229" alt="4vms" src="https://github.com/user-attachments/assets/0caa608e-30d7-4676-afc1-1090700891ca" />
All 4 virtual machines created in VirtualBox: Kali Attack, Wazuh, Splunk and Windows Target.

#### Kali Attack — VM Configuration
<img width="630" height="721" alt="Kali-vm-config" src="https://github.com/user-attachments/assets/50cf1da2-918b-4364-bd2f-dba4f211d579" />
- OS: Ubuntu 64-bit (Kali Linux)
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

 #### Windows Target — VM Configuration
<img width="601" height="772" alt="windows-target-vm-config" src="https://github.com/user-attachments/assets/dec23bf9-7865-4d8b-b31d-ff66ff53d3a3" />
- OS: Windows 11 64-bit
- RAM: 4096 MB | CPUs: 2
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Wazuh — VM Configuration
<img width="608" height="714" alt="wazuh-vm-config" src="https://github.com/user-attachments/assets/f89c0104-49f0-4a9f-bdcf-c1d9d6681467" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

#### Splunk — VM Configuration
<img width="604" height="731" alt="splunk-vm-config" src="https://github.com/user-attachments/assets/9d38cc56-3ff7-43dd-a7a0-b00970034a9a" />
- OS: Ubuntu 64-bit
- RAM: 6144 MB | CPUs: 4
- Network: Adapter 1 NAT + Adapter 2 Internal Network

### 2. Network — Configuration Adapter 1 NAT
<img width="561" height="307" alt="network-adp-nat" src="https://github.com/user-attachments/assets/813c7764-3bc1-4e15-ac20-30c63e51d2e2" />

Adapter 1 is set to NAT mode to provide internet access inside each VM. 
This is necessary to be able to update the system, install packages and 
download tools throughout the project. Without this adapter, the VMs would 
be completely isolated with no ability to reach external repositories or 
download software.

### 3. Network — Configuration Adapter 2 Internal Network
<img width="563" height="305" alt="network-adp-internal" src="https://github.com/user-attachments/assets/74662cd1-8221-40b9-bbb9-b95ea05a9147" />

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
<img width="648" height="283" alt="kali-config-stat" src="https://github.com/user-attachments/assets/f9fad031-0ab7-498f-9b39-6657cfa8648d" />
To assign a static IP in Linux, the network configuration file must be edited 
manually via terminal. The file /etc/network/interfaces is modified to set 
a fixed IP address, subnet mask and interface for the Internal Network adapter.
<img width="698" height="515" alt="kali-config-w" src="https://github.com/user-attachments/assets/cdca544b-4f1c-4c58-9b31-a49029799678" />
Once the configuration is applied and the network service restarted, the 
command ip a is used to verify that the static IP has been correctly assigned 
to the interface.


#### Target Windows
<img width="501" height="632" alt="windows-config-stat" src="https://github.com/user-attachments/assets/1866db66-2b56-4778-9956-0aaf455fc0fb" />
To assign a static IP in Windows 11, the network adapter settings are 
accessed through Control Panel. The Internal Network adapter is configured 
with a fixed IP address and subnet mask under IPv4 properties.
<img width="983" height="509" alt="windows-config-w" src="https://github.com/user-attachments/assets/25edc3b0-f9d7-4da4-b66c-a7ad0545dd3a" />
The command ipconfig in CMD confirms the static IP has been successfully 
assigned to the Internal Network adapter.


#### Wazuh
<img width="1281" height="799" alt="image" src="https://github.com/user-attachments/assets/f2fa30a3-a8bc-46d9-9929-e84e75b4e67f" />
To assign a static IP in Ubuntu Desktop, the network settings are accessed 
through the GUI. The Internal Network adapter is set to Manual mode with a 
fixed IP address and subnet mask.
<img width="1024" height="458" alt="wazuh-config-w" src="https://github.com/user-attachments/assets/7b011143-a163-479b-9b78-9aca1e5f2930" />
The command ip a confirms the static IP has been successfully applied 
to the enp0s8 interface.


#### Splunk
<img width="1285" height="801" alt="wazuh-config-stat" src="https://github.com/user-attachments/assets/68ceeb0d-d9fd-4e9a-acd7-53177bd05ce0" />

To assign a static IP in Ubuntu Desktop, the network settings are accessed 
through the GUI. The Internal Network adapter is set to Manual mode with a 
fixed IP address and subnet mask.
<img width="1038" height="573" alt="splunk-config-w" src="https://github.com/user-attachments/assets/a9cfaa35-a82a-460a-820b-9ed94cd71a49" />
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
<img width="505" height="189" alt="ping-windows" src="https://github.com/user-attachments/assets/bc852a86-a05c-444e-895c-019e57e1429d" />
Successful ping to 192.168.10.20 confirming connectivity with the Windows Target machine.

#### Linux > Wazuh
<img width="508" height="185" alt="ping-wazuh" src="https://github.com/user-attachments/assets/decb153d-e915-4ac6-95f9-3ca902486f47" />
Successful ping to 192.168.10.30 confirming connectivity with the Wazuh Manager.

#### Linux > Splunk
<img width="508" height="186" alt="ping-splunk" src="https://github.com/user-attachments/assets/c8e96787-6322-4641-86a4-cdfd9b6bdfa1" />
Successful ping to 192.168.10.40 confirming connectivity with the Splunk SIEM.

