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
| Windows 11 Pro | 192.168.10.20 |
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

### VM 2 — Windows 11 Pro (Target)

| Setting | Value |
|---------|-------|
| OS | Windows 11 Pro (64-bit) |
| RAM | 4 GB |
| CPU | 2 cores |
| Disk | 60 GB (dynamically allocated) |
| Adapter 1 | Internal Network — `SOC-Homelab` |
| Role | Attack target + Wazuh Agent host |

**Static IP configuration** (Control Panel → Network & Sharing → Adapter settings):
```
Interface: Ethernet (Adapter 1)
IP:        192.168.10.20
Netmask:   255.255.255.0
```

---

### VM 3 — Ubuntu Server — Wazuh Manager

| Setting | Value |
|---------|-------|
| OS | Ubuntu 26.04 LTS Server |
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
| OS | Ubuntu 26.04 LTS Server |
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
ping -c 3 192.168.10.20   # Windows 11
ping -c 3 192.168.10.30   # Wazuh Manager
ping -c 3 192.168.10.40   # Splunk
```

**From Wazuh Manager — ping target and SIEM:**
```bash
ping -c 3 192.168.10.20   # Windows 11
ping -c 3 192.168.10.40   # Splunk
```

## Screenshots

| Screenshot | Description |
|------------|-------------|
| `screenshots/phase1/kali-ip.png` | `ip a` output showing 192.168.10.10 |
| <img width="614" height="144" alt="image" src="https://github.com/user-attachments/assets/b0f6c224-c675-4afa-9745-9529215cc689" /> | `ipconfig` showing 192.168.10.20 |
| <img width="812" height="110" alt="image" src="https://github.com/user-attachments/assets/fec37eca-7cd1-478d-9ad0-863108745add" /> | `ip a` showing 192.168.10.30 |
| <img width="810" height="110" alt="image" src="https://github.com/user-attachments/assets/64879143-ef23-4c95-acb8-20c46df59951" /> | `ip a` showing 192.168.10.40 |
| `screenshots/phase1/ping-test.png` | Ping from Kali to all other VMs |

---

*Next: [Phase 2 — Wazuh Deployment](phase2-wazuh.md)*
