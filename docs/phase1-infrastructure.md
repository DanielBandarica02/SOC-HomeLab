Phase 1 — Infrastructure Setup
Overview
This phase covers the full setup of the virtualized SOC lab environment. All machines run inside VirtualBox on a single host using an isolated internal network, simulating a real enterprise segment without exposing anything to the outside world.

Host Machine Specs
ComponentSpecCPUIntel Core i7-14700KFRAM32 GBGPUNVIDIA RTX 5070HypervisorVirtualBox (latest)

Network Design
All VMs communicate over a VirtualBox Internal Network named SOC-Homelab.
Machines that need internet access (to install packages) also have a NAT adapter.
VMInternal IPNAT (internet)Kali Linux192.168.10.10✅ YesWindows 11 Pro192.168.10.20❌ NoUbuntu — Wazuh Manager192.168.10.30✅ YesUbuntu — Splunk192.168.10.40✅ Yes
Data flow:
Windows 11 Pro → Wazuh Agent → Wazuh Manager → Splunk HEC → Splunk SIEM

Virtual Machines
VM 1 — Kali Linux (Attacker)
SettingValueOSKali Linux (latest, 64-bit)RAM4 GBCPU2 coresDisk50 GB (dynamically allocated)Adapter 1NATAdapter 2Internal Network — SOC-HomelabRoleAttack machine (Nmap, Hydra, Metasploit, Burp Suite)
Static IP configuration (/etc/network/interfaces or via NetworkManager):
Interface: eth1 (Adapter 2)
IP:        192.168.10.10
Netmask:   255.255.255.0
Gateway:   (none — internal only)

VM 2 — Windows 11 Pro (Target)
SettingValueOSWindows 11 Pro (64-bit)RAM4 GBCPU2 coresDisk60 GB (dynamically allocated)Adapter 1Internal Network — SOC-HomelabRoleAttack target + Wazuh Agent host
Static IP configuration (Control Panel → Network & Sharing → Adapter settings):
Interface: Ethernet (Adapter 1)
IP:        192.168.10.20
Netmask:   255.255.255.0
Gateway:   (none)
DNS:       (none)

Note: Windows Defender and the firewall are left enabled intentionally. This makes the environment realistic — the goal is to detect attacks through logs, not to make attacks easier.


VM 3 — Ubuntu Server — Wazuh Manager
SettingValueOSUbuntu 26.04 LTS ServerRAM8 GBCPU2 coresDisk50 GB (dynamically allocated)Adapter 1NATAdapter 2Internal Network — SOC-HomelabRoleWazuh Manager (EDR backend) + alert forwarding to Splunk via HEC
Static IP configuration (/etc/netplan/00-installer-config.yaml):
yamlnetwork:
  version: 2
  ethernets:
    enp0s3:           # Adapter 1 - NAT (DHCP for internet)
      dhcp4: true
    enp0s8:           # Adapter 2 - Internal
      dhcp4: false
      addresses:
        - 192.168.10.30/24
Apply with:
bashsudo netplan apply

VM 4 — Ubuntu Server — Splunk
SettingValueOSUbuntu 26.04 LTS ServerRAM8 GBCPU2 coresDisk60 GB (dynamically allocated)Adapter 1NATAdapter 2Internal Network — SOC-HomelabRoleSplunk SIEM (dashboards, SPL queries, correlation rules)
Static IP configuration (/etc/netplan/00-installer-config.yaml):
yamlnetwork:
  version: 2
  ethernets:
    enp0s3:           # Adapter 1 - NAT (DHCP for internet)
      dhcp4: true
    enp0s8:           # Adapter 2 - Internal
      dhcp4: false
      addresses:
        - 192.168.10.40/24
Apply with:
bashsudo netplan apply

Connectivity Verification
Once all VMs are running and IPs are assigned, verify connectivity from each machine:
From Kali — ping all hosts:
bashping -c 3 192.168.10.20   # Windows 11
ping -c 3 192.168.10.30   # Wazuh Manager
ping -c 3 192.168.10.40   # Splunk
From Wazuh Manager — ping target and SIEM:
bashping -c 3 192.168.10.20   # Windows 11
ping -c 3 192.168.10.40   # Splunk

Note: Windows 11 blocks ICMP by default. If the ping fails, verify the internal adapter IP is set correctly using ipconfig inside the Windows VM. You can also temporarily allow ICMP via Windows Firewall to test — but it is not required for the lab to function.


Resource Allocation Summary
VMRAMCPUDiskKali Linux4 GB2 cores50 GBWindows 11 Pro4 GB2 cores60 GBUbuntu Wazuh8 GB2 cores50 GBUbuntu Splunk8 GB2 cores60 GBTotal24 GB8 cores220 GB
With 32 GB of host RAM, 8 GB remains for the host OS — enough to run everything simultaneously without issues.

Phase 1 — Completion Checklist

 VirtualBox installed on host
 Internal Network SOC-Homelab created
 Kali Linux VM created and configured (NAT + Internal)
 Windows 11 Pro VM created and configured (Internal only)
 Ubuntu Wazuh VM created and configured (NAT + Internal)
 Ubuntu Splunk VM created and configured (NAT + Internal)
 Static IPs assigned on all VMs
 Connectivity verified between all machines
 Screenshots taken of each VM running and showing correct IP (ip a / ipconfig)


Screenshots

(Add screenshots here as you complete each step)

ScreenshotDescriptionscreenshots/phase1/kali-ip.pngip a output showing 192.168.10.10screenshots/phase1/win11-ip.pngipconfig showing 192.168.10.20screenshots/phase1/wazuh-ip.pngip a showing 192.168.10.30screenshots/phase1/splunk-ip.pngip a showing 192.168.10.40screenshots/phase1/ping-test.pngPing from Kali to all other VMs
