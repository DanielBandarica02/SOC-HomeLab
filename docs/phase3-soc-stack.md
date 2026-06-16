# Phase 3 — SOC Stack Deployment (Wazuh + Splunk)
 
## Overview
 
The SOC management plane is deployed within a dedicated VLAN 99 utilizing two Ubuntu Server VMs operating in tandem: Wazuh serves as the central manager for distributed endpoint EDR agents, while Splunk Enterprise acts as the central SIEM, ingest-parsing alerts and raw telemetry for unified threat hunting, dashboarding, and incident response.
 
---
 
## Environment
 
| Component                    | Version          | Host                              |
| ---------------------------- | ---------------- | --------------------------------- |
| Wazuh (manager + indexer + dashboard) | 4.14.5    | Ubuntu Server 24.04 LTS VM        |
| Splunk Enterprise            | 10.4.0           | Ubuntu Server 24.04 LTS VM        |
| Wazuh → Splunk integration   | Custom Integrator + curl → HEC 8088 | Ubuntu Server 22.04 LTS VM |
 
---
 
## Architecture
 
```mermaid
flowchart LR
    subgraph DATA[Data-plane VLANs]
        EP1[Endpoints<br/>VLAN 10, 20]
        PF[pfSense + Suricata<br/>10.10.99.1]
    end
    subgraph SOC[VLAN 99 — SOC Management]
        WZ[Wazuh Manager<br/>10.10.99.10<br/>Manager :1514 · API :55000<br/>Dashboard :443]
        SP[Splunk Enterprise<br/>10.10.99.20<br/>Web :8000 · HEC :8088 · UF :9997]
    end
    EP1 -- "Wazuh agent :1514" --> PF --> WZ
    PF -- "Suricata eve.json → HEC :8088" --> SP
    WZ -- "Filebeat → HEC :8088<br/>alerts.json forward" --> SP
```
 
---
 
## Deployment
 
### SOC VMs provisioning
 
Two Ubuntu Server VMs were created in VirtualBox attached to Internal Network `vlan99-soc`.
 
| VM              | vCPU | RAM   | Disk        | IP            |
| --------------- | ---- | ----- | ----------- | ------------- |
| SOC-99-Wazuh    | 4    | 8 GB  | 60 GB | 10.10.99.10   |
| SOC-99-Splunk   | 2    | 4 GB  | 40 GB | 10.10.99.20   |
 
### Ubuntu Server Network Configuration
 
| Screenshot | Description |
| ---------- | ----------- |
| ![Wazuh Network IPv4](../screenshots/phase3/01-wazuh-ipv4-configuration.png) | Wazuh Network IPv4 Configuration |
| ![Splunk Network IPv4](../screenshots/phase3/02-splunk-ipv4-configuration.png)| Splunk Network IPv4 Configuration |

---

### Wazuh Integration
 
The Wazuh Manager was installed on Ubuntu Server 24.04 using the all-in-one installation script, which deploys the Manager, Indexer, and Dashboard in a single-node configuration:
 
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
 
The `-a` flag performs a complete all-in-one installation. Upon completion, the installer outputs the admin credentials required to access the dashboard.
 
The dashboard is accessible at `https://10.10.99.10:443`.

![Wazuh Dashboard home](../screenshots/phase3/03-wazuh-dashboard-home.png)

---

### Splunk Enterprise

Splunk Enterprise 10.4.0 was installed on Ubuntu Server 24.04 using the official `.deb` package downloaded from the Splunk portal:

```bash
sudo dpkg -i splunk-10.4.0-f798d4d49089-linux-amd64.deb
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

The dashboard is accessible at `http:/10.10.99.20:8000`.

![Splunk Web home](../screenshots/phase3/04-splunk-web-home.png)

## HTTP Event Collector (HEC)

The HEC was configured in Splunk to receive Wazuh alerts over HTTP:

- **Settings → Data Inputs → HTTP Event Collector → Global Settings** — HEC enabled, SSL disabled, port `8088`
- A dedicated token was created with the following settings:
  - Name: `Wazuh_Alerts`
  - Source type: `wazuh`
  - Default index: `wazuh`

A dedicated index was created to isolate Wazuh data:

- **Settings → Indexes → New Index**
- Index name: `wazuh`

### Wazuh Integration

The Wazuh Manager was configured to forward all alerts to the Splunk HEC by adding the following block to `/var/ossec/etc/ossec.conf`:

```xml
<integration>
  <name>custom-splunk-hec</name>
  <hook_url>http://192.168.10.40:8088/services/collector/event</hook_url>
  <api_key>4a0b64e1-5e39-449f-a88e-63d0d3159e89</api_key>
  <alert_format>json</alert_format>
  <level>0</level>
</integration>
```

The Wazuh Manager was restarted to apply the changes:

```bash
sudo systemctl restart wazuh-manager
```

---

## Troubleshooting & Lessons Learned

### HEC Integration — Alert Forwarding Boundary & Block Placement

During the initial validation of the Splunk integration using an SSH brute-force attack (via Hydra), no security events or alerts were appearing in Splunk Search, despite the HEC token being correctly configured and port 8088 being open. The root cause was twofold:

1. **Syntax Placement:** The `<integration>` block in `ossec.conf` was mistakenly placed inside a duplicate `<ossec_config>` tag that the Wazuh Manager completely ignored.
2. **Alert Filtering Threshold:** By default, the Wazuh integration engine applies strict filtering thresholds for forwarding events. Without an explicit instruction, lower-level or baseline security telemetry was discarded before leaving the manager.

**Solution:** 
Ensured the `<integration>` block was placed inside the primary, single valid `<ossec_config>` block (immediately before the closing `</ossec_config>` tag) and explicitly forced the engine to forward all telemetry by setting `<level>0</level>` within the integration parameters.

```xml
<integration>
  <name>custom-splunk-hec</name>
  <hook_url>http://192.168.10.40:8088/services/collector/event</hook_url>
  <api_key>4a0b64e1-5e39-449f-a88e-63d0d3159e89</api_key>
  <alert_format>json</alert_format>
  <level>0</level>  <!-- Crucial to forward all severities and host events -->
</integration>
```

---
 
## Result
 
- Two Ubuntu Server VMs operational in VLAN 99 (`10.10.99.10` Wazuh, `10.10.99.20` Splunk)
- Wazuh 4.14 all-in-one stack running (manager + indexer + dashboard) at `https://10.10.99.10`
- Splunk Enterprise 10.4 running at `http://10.10.99.20:8000`
- HEC enabled on Splunk port 8088 with `wazuh-alerts` token issued
  
---
 
## Screenshots
 
| Screenshot | Description |
| ---------- | ----------- |
| [![Wazuh services running](../screenshots/phase3/02-wazuh-services-active.png)](../screenshots/phase3/02-wazuh-services-active.png) | `systemctl status wazuh-manager wazuh-indexer wazuh-dashboard` all active |
| [![Splunk HEC token](../screenshots/phase3/04-splunk-hec-token.png)](../screenshots/phase3/04-splunk-hec-token.png) | `wazuh-alerts` HEC token created and enabled |
 
---
 
*Previous: [Phase 2 — Network Backbone (pfSense)](phase2-network-backbone.md)*  
*Next: [Phase 4 — Corporate Environment & Endpoint Telemetry](phase4-corporate-env.md)*
