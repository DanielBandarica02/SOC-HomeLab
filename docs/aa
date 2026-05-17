# Phase 1: Infrastructure Setup & Network Topology

## 1. Project Objective & Isolation Strategy
The core objective of this phase is to deploy a completely isolated, secure, and controlled virtual laboratory sandbox environment. When performing adversarial simulations (malware execution, network attacks, exploitation), containment is critical. 

By designing a custom virtual network layer, we guarantee that live attack payloads executed from our offensive platform remain fully contained inside the lab boundary, avoiding any impact on the external domestic production network.



---

## 2. Logical Network Architecture
The environment utilizes a single internal IP data plane based on the private Class C subnet: **`192.168.10.0/24`**. All Virtual Machines (VMs) are configured with static IP addresses to preserve telemetry consistency and rule correlation integrity.

### Host Deployment Matrix

| Virtual Machine | Operating System | Role inside the Architecture | IP Address | Resource Allocation |
| :--- | :--- | :--- | :--- | :--- |
| **Kali Linux** | Kali Rolling x64 | Adversarial Simulation Platform (Attacker) | `192.168.10.5` | 4 GB RAM / 2 vCPUs |
| **Windows 11 Pro** | Win 11 Enterprise (Eval) | Target Endpoint Monitoring Environment | `192.168.10.10` | 4 GB RAM / 2 vCPUs |
| **Wazuh Manager** | Ubuntu Server 22.04 LTS | Central EDR Engine & Threat Alert Compiler | `192.168.10.15` | 8 GB RAM / 2 vCPUs |
| **Splunk Enterprise**| Ubuntu Server 22.04 LTS | SIEM Analytics Core & Dashboarding Platform | `192.168.10.20` | 8 GB RAM / 2 vCPUs |

---

## 3. Hypervisor Virtual Network Mapping
To facilitate communication across components while enforcing containment, a custom VirtualBox **NAT Network** is used. This allows outbound WAN access for package installation and security updates via an implicit routing gateway, while enabling isolated internal cross-talk.

### Step-by-Step Configuration:
1. Open **Oracle VM VirtualBox Manager**.
2. Navigate to **File** ➔ **Tools** ➔ **Network Manager**.
3. Under the **NAT Networks** tab, click **Create**.
4. Define the structural parameters:
   * **Network Name:** `SOC-Network`
   * **Network CIDR:** `192.168.10.0/24`
   * **Supports DHCP:** *Unchecked (Disabled)* - Essential to prevent dynamic IP reassignment from breaking EDR/SIEM pipelines.

> 📸 **[SCREENSHOT PLACEHOLDER 1: VirtualBox Network Manager Settings]**
> *Take a screenshot of your VirtualBox Preferences/Network Manager showing the "SOC-Network" setup, confirming the CIDR block and that DHCP is disabled.*

---

## 4. Virtual Machine Network Adapter Configuration
Every virtual machine deployed in this lab must have its virtual interface explicitly attached to the custom network structure.

1. Select the Virtual Machine and click **Settings** ➔ **Network**.
2. Set **Attached to:** `NAT Network`.
3. Set **Name:** `SOC-Network`.
4. Expand **Advanced** and ensure the following are configured:
   * **Adapter Type:** `Intel PRO/1000 MT Desktop (82540EM)`
   * **Promiscuous Mode:** `Allow All` *(This is mandatory for Phase 4 to allow Snort to ingest raw network traffic).*
   * **Cable Connected:** *Checked*.

> 📸 **[SCREENSHOT PLACEHOLDER 2: Individual VM Network Settings]**
> *Take a screenshot of the Network tab of one of your VMs (e.g., Windows 11) showing the NAT Network connection and the Advanced tab with Promiscuous Mode: Allow All.*

---

## 5. Base Operating System Configurations & Interfaces

### 5.1. Kali Linux (Attacker Platform)
Post-installation, network interface management is moved away from dynamic DHCP to enforce a persistent network footprint.

```bash
sudo nano /etc/network/interfaces
