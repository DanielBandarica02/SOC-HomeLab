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
Machines that need internet access (to install packages) also have a **NAT adapter**.

| VM | Internal IP | NAT (internet) |
|----|-------------|----------------|
| Kali Linux | 192.168.10.10 |  Yes |
| Windows 11 Pro | 192.168.10.20 |  No |
| Ubuntu — Wazuh Manager | 192.168.10.30 |  Yes |
| Ubuntu — Splunk | 192.168.10.40 |  Yes |

**Data flow:**  
`Windows 11 Pro → Wazuh Agent → Wazuh Manager → Splunk HEC → Splunk SIEM`

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

**Static IP configuration** (`/etc/network/interfaces` or via NetworkManager):
```
Interface: eth1 (Adapter 2)
IP:        192.168.10.10
Netmask:   255.255.255.0
Gateway:   (none — internal only)
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
Gateway:   (none)
DNS:       (none)
```

> **Note:** Windows Defender and the firewall are left **enabled** intentionally. This makes the environment realistic — the goal is to detect attacks through logs, not to make attacks easier.

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
  ethernets:
    enp0s3:           # Adapter 1 - NAT (DHCP for internet)
      dhcp4: true
    enp0s8:           # Adapter 2 - Internal
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
  ethernets:
    enp0s3:           # Adapter 1 - NAT (DHCP for internet)
      dhcp4: true
    enp0s8:           # Adapter 2 - Internal
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

> **Note:** Windows 11 blocks ICMP by default. If the ping fails, verify the internal adapter IP is set correctly using `ipconfig` inside the Windows VM. You can also temporarily allow ICMP via Windows Firewall to test — but it is not required for the lab to function.

---

## Resource Allocation Summary

| VM | RAM | CPU | Disk |
|----|-----|-----|------|
| Kali Linux | 4 GB | 2 cores | 50 GB |
| Windows 11 Pro | 4 GB | 2 cores | 60 GB |
| Ubuntu Wazuh | 8 GB | 2 cores | 50 GB |
| Ubuntu Splunk | 8 GB | 2 cores | 60 GB |
| **Total** | **24 GB** | **8 cores** | **220 GB** |

With 32 GB of host RAM, 8 GB remains for the host OS — enough to run everything simultaneously without issues.

## Screenshots

> *(Add screenshots here as you complete each step)*

| Screenshot | Description |
|------------|-------------|
| `screenshots/phase1/kali-ip.png` | `ip a` output showing 192.168.10.10 |
| `screenshots/phase1/win11-ip.png` | `ipconfig` showing 192.168.10.20 |
| `screenshots/phase1/wazuh-ip.png` | `ip a` showing 192.168.10.30 |
| `screenshots/phase1/splunk-ip.png` | `ip a` showing 192.168.10.40 |
| `screenshots/phase1/ping-test.png` | Ping from Kali to all other VMs |

---

*Next: [Phase 2 — Wazuh Deployment](phase2-wazuh.md)*
