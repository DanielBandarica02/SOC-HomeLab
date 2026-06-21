# Prerequisites

## Host hardware

| Resource | Minimum                                                | Recommended     |
| -------- | ------------------------------------------------------ | --------------- |
| CPU      | 4 cores with virtualization (VT-x / AMD-V) enabled     | 8+ cores        |
| RAM      | 24 GB                                                  | 32+ GB          |
| Disk     | 300 GB free on SSD                                     | 500+ GB SSD     |

## Host software

- **Oracle VirtualBox**
- SSH client.
- Modern browser (for Wazuh Dashboard and pfSense Web UI).

## ISOs used

| ISO                        | Source                                                      |
| -------------------------- | ----------------------------------------------------------- |
| pfSense CE                 | https://www.pfsense.org/download/                           |
| Windows Server 2022 Eval   | https://www.microsoft.com/en-us/evalcenter/                 |
| Windows 11 Pro Eval        | https://www.microsoft.com/en-us/software-download/windows11 |
| Ubuntu Server 24.04 LTS    | https://ubuntu.com/download/server                          |
| Ubuntu Desktop 24.04 LTS   | https://ubuntu.com/download/desktop                         |
| Kali Linux                 | https://www.kali.org/get-kali/                              |

## VM resource allocation
 
These are the per-VM allocations I'll configure in VirtualBox. The numbers below are the minimum I consider healthy under normal lab usage — running below this is possible but trades stability for headroom on the host.
 
| VM                       | vCPU | RAM    | Disk    |
| ------------------------ | :--: | :----: | :-----: |
| pfSense CE               |   2  |  2 GB  |  20 GB  |
| WinSrv2022 (AD DC)       |   2  |  4 GB  |  60 GB  |
| Win11-Corp               |   2  |  4 GB  |  60 GB  |
| Win11-Dev                |   2  |  4 GB  |  60 GB  |
| Ubuntu-Dev               |   2  |  4 GB  |  40 GB  |
| Kali                     |   2  |  4 GB  |  40 GB  |
| Ubuntu-Server (Wazuh)    |   4  |  8 GB  |  80 GB  |

 

## VirtualBox network topology

Since VirtualBox doesn't implement 802.1Q VLAN tagging, I translate the diagram's architecture into **Internal Networks** (hypervisor-level isolated networks visible only between VMs that explicitly share them).

| Internal Network    | Purpose                       | VMs attached                            |
| ------------------- | ----------------------------- | --------------------------------------- |
| `internal-wan`          | NAT → Internet                | pfSense (WAN interface)                 |
| `internal-vlan10-corp`  | VLAN 10 — Corporate           | pfSense · WinSrv2022 · Win11-Corp       |
| `internal-vlan20-dev`   | VLAN 20 — Software Dev        | pfSense · Win11-Dev · Ubuntu-Dev        |
| `internal-vlan66-dmz`   | VLAN 66 — Attacker DMZ        | pfSense · Kali                          |
| `internal-vlan99-soc`   | VLAN 99 — SOC Management      | pfSense · Ubuntu-Server (Wazuh)         |

**pfSense will have 5 network adapters:** one per Internal Network. It will be the only routing point between segments — exactly like a physical firewall in a real corporate network.

## IP map

| VM                       | IP            | Segment   | Role                                |
| ------------------------ | ------------- | --------- | ----------------------------------- |
| pfSense (VLAN 10 gw)     | 10.10.10.1    | VLAN 10   | Gateway / Firewall / IDS / VPN      |
| pfSense (VLAN 20 gw)     | 10.10.20.1    | VLAN 20   | Gateway / Firewall / IDS            |
| pfSense (VLAN 66 gw)     | 10.10.66.1    | VLAN 66   | Gateway / Firewall / IDS            |
| pfSense (VLAN 99 gw)     | 10.10.99.1    | VLAN 99   | Gateway / Firewall / IDS            |
| WinSrv2022 (AD DC)       | 10.10.10.10   | VLAN 10   | Active Directory Domain Controller  |
| Win11-Corp               | 10.10.10.20   | VLAN 10   | Corporate workstation               |
| Win11-Dev                | 10.10.20.10   | VLAN 20   | Dev workstation (Windows)           |
| Ubuntu-Dev               | 10.10.20.20   | VLAN 20   | Dev workstation (Linux)             |
| Kali                     | 10.10.66.10   | VLAN 66   | Adversary emulation                 |
| Ubuntu-Server (Wazuh)    | 10.10.99.10   | VLAN 99   | SIEM / EDR all-in-one               |

### OpenVPN tunnel 

| Resource         | Value              | Notes                                                                     |
| ---------------- | ------------------ | ------------------------------------------------------------------------- |
| Server endpoint  | `10.10.10.1:1194`  | pfSense, bound to the VLAN 10 gateway interface, UDP                       |
| Client pool      | `10.10.50.0/24`    | Logical subnet — no VirtualBox Internal Network required                  |
| Pushed routes    | `10.10.20.0/24`    | Tunnel clients reach VLAN 20 through pfSense after authenticating         |
| Allowed services | RDP 3389 · SSH 22  | Enforced by pfSense firewall rules on the OpenVPN interface (manual)      |

**Reasoning:** VLAN 10 ↔ VLAN 20 is intentionally denied at the firewall. The only sanctioned crossing is the OpenVPN tunnel, which adds authentication + logging on top of the network controls.


