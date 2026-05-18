# Phase 1 — Infrastructure Setup

## Overview

This phase covers the full setup of the virtualized SOC lab environment. All machines run inside VirtualBox on a single host using an isolated internal network, simulating a real enterprise segment without exposing anything to the outside world.

---

## Host Machine Specs

| Component | Spec |
|-----------|------|
| CPU | Intel Core i7-14700KF |
| RAM | 32 GB |
| GPU | NVIDIA RTX 5070 |
| Hypervisor | VirtualBox (latest) |

---

## Network Design

All VMs communicate over a VirtualBox **Internal Network** named `SOC-Homelab`.  


| VM | Internal IP | 
|----|-------------|
| Kali Linux | 192.168.10.10 |
| Ubuntu Desktop | 192.168.10.20 |
| Ubuntu — Wazuh Manager | 192.168.10.30 |
| Ubuntu — Splunk | 192.168.10.40 |

**Data flow:**  
Windows 11 Pro → Wazuh Agent → Wazuh Manager → Splunk HEC → Splunk SIEM

---

## Virtual Machines

### VM 1 — Kali Linux (Attacker)

| Setting | Value |
|---------|-------|
| OS | Kali Linux (latest, 64-bit) |
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | NAT |
| Adapter 2 | Internal Network — `SOC-Homelab` |
| Role | Attack machine (Nmap, Hydra, Metasploit, Burp Suite) |

**Static IP configuration** (`/etc/network/interfaces`):
```
Interface: eth1 (Adapter 2)
IP:        192.168.10.10
Netmask:   255.255.255.0
```

---

### VM 2 – Ubuntu Desktop 24.04 (Target)

| Setting | Value |
|---|---|
| OS | Ubuntu Desktop 24.04.3 (amd64) |
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | Internal Network – `SOC-Homelab` |
| Role | Attack target + Wazuh Agent host |

**Static IP configuration** (Settings -> Network -> Wired):
```text
Interface: enp0s3 (or similar)
IP:        192.168.10.20
Netmask:   255.255.255.0 / CIDR: 24
```
---

### VM 3 — Ubuntu Server — Wazuh Manager

| Setting | Value |
|---------|-------|
| OS | Ubuntu 24.04 LTS Server |
| RAM | 8 GB |
| CPU | 2 cores |
| Disk | 50 GB (dynamically allocated) |
| Adapter 1 | NAT |
| Adapter 2 | Internal Network — `SOC-Homelab` |
| Role | Wazuh Manager (EDR backend) + alert forwarding to Splunk via HEC |

**Static IP configuration** (`/etc/netplan/00-installer-config.yaml`):
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.10.30/24
```

Apply with:
```bash
sudo netplan apply
```

---

### VM 4 — Ubuntu Server — Splunk

| Setting | Value |
|---------|-------|
| OS | Ubuntu 24.04 LTS Server |
| RAM | 8 GB |
| CPU | 2 cores |
| Disk | 60 GB (dynamically allocated) |
| Adapter 1 | NAT |
| Adapter 2 | Internal Network — `SOC-Homelab` |
| Role | Splunk SIEM (dashboards, SPL queries, correlation rules) |

**Static IP configuration** (`/etc/netplan/00-installer-config.yaml`):
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.10.40/24
```

Apply with:
```bash
sudo netplan apply
```

---

## Connectivity Verification

Once all VMs are running and IPs are assigned, verify connectivity from each machine:

**From Kali — ping all hosts:**
```bash
ping -c 1 192.168.10.20   # Ubuntu Desktop
ping -c 1 192.168.10.30   # Wazuh Manager
ping -c 1 192.168.10.40   # Splunk
```
![Ping Kali-Ubuntu](../screenshots/phase1/pingkaliubu.png)

![Ping Kali-Wazuh](../screenshots/phase1/pingkaliwaz.png)

![Ping Kali-Splunk](../screenshots/phase1/pingkalisplu.png)


**From Wazuh Manager — ping target and SIEM:**
```bash
ping -c 1 192.168.10.20   # Windows 11
ping -c 1 192.168.10.40   # Splunk
```
![Ping Wazuh-Ubuntu](../screenshots/phase1/pingwazuhubu.png)

![Ping Wazuh-Splunk](../screenshots/phase1/pingwazuhsplun.png)


## Screenshots

| Screenshot | Description |
|------------|-------------|
| ![Kali IP](../screenshots/phase1/ipkali.png) | `ip a` output showing 192.168.10.10 |
| ![Ubuntu IP](../screenshots/phase1/ipubuntu.png) | `ip a` showing 192.168.10.20 |
| ![Wazuh IP](../screenshots/phase1/ipwazuh.png) | `ip a` showing 192.168.10.30 |
| ![Splunk IP](../screenshots/phase1/ipsplunk.png) | `ip a` showing 192.168.10.40 |

---

*Next: [Phase 2 — Wazuh Deployment](phase2-wazuh.md)*
