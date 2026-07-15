# Phase 7 — SOC Platform Integration
 
## Overview
 
Phase 7 extends the SOC HomeLab from a SIEM-only detection environment (Wazuh) to a fully integrated SOC platform combining detection, case management, threat intelligence enrichment, and network-based intrusion detection. The goal of this phase is to reproduce the operational workflow of a small-to-medium production SOC using entirely open-source tooling.
 
Prior phases established the following capabilities:
- Network infrastructure with pfSense-based VLAN segmentation (Phase 1)
- Wazuh SIEM deployment with agents and dashboards (Phase 2)
- Adversary simulation environment in Kali Linux (Phase 3)
- Attack scenario execution and documentation (Phase 4)
- Custom detection rules mapped to MITRE ATT&CK (Phase 5)
- L1 incident report following industry conventions (Phase 6)
Phase 7 adds the operational tools that turn detection into response.
 
---

## Components
 
The Phase 7 stack consists of four tools deployed across two hosts.
 
### On pfSense (native package)
 
**Suricata** — Network-based Intrusion Detection System (IDS). Deployed as a native pfSense package with sensors on the Corp, Dev, and Adversary DMZ VLANs. Suricata inspects network traffic at packet level, matches against the Emerging Threats Open ruleset, and generates EVE JSON alerts that are consumed by the Wazuh Agent installed on pfSense. Suricata operates in Legacy Mode (IDS-only, no packet blocking) to preserve the attack traffic for detection engineering documentation.
 
### On a new dedicated VM (`soc-platform`, 10.10.99.20)
 
**TheHive** — Security incident response platform providing case management, task tracking, and observable management. Wazuh alerts of severity level 10 or higher trigger the automatic creation of a TheHive case via webhook integration. TheHive is deployed as the entry point for the response layer of the SOC.
 
**Cortex** — Analysis and enrichment engine that integrates with TheHive. Cortex hosts analyzers (WHOIS, DNSDB, VirusTotal, AbuseIPDB, MISP lookup, and others) that are invoked from TheHive cases to enrich observables automatically. Cortex is deployed alongside TheHive as part of the StrangeBee ecosystem.
 
**MISP** — Malware Information Sharing Platform for threat intelligence management. MISP aggregates IoC feeds from public sources (CIRCL, Botvrij, Malware Bazaar) and provides context lookup for observables surfaced during case investigation. MISP integrates with Cortex as an analyzer target, allowing cases to enrich their IoCs with threat intelligence context automatically.
 
---

## Data flow
 
The Phase 7 stack implements the following end-to-end workflow for a typical alert:
 
1. **Network traffic reaches pfSense** from any VLAN. Suricata sensors on VLAN 10, 20, and 66 inspect the traffic against the Emerging Threats Open ruleset.
2. **Suricata generates alerts** in EVE JSON format when signatures match. Alerts are written to `/var/log/suricata/suricata_<interface>/eve.json` on pfSense.
3. **Wazuh Agent on pfSense** reads the EVE JSON file continuously and forwards each alert to the Wazuh Manager on 10.10.99.10.
4. **Wazuh Manager processes the alerts** using its native Suricata ruleset (rules 86000-86999), correlating them with logs from other sources (Windows and Linux endpoint agents, pfSense filterlog).
5. **Custom Wazuh rules apply on top** of the Suricata-tagged events, adding organisation-specific context or promoting patterns to higher severity.
6. **Alerts of severity level 10 or higher trigger a webhook** to TheHive on the `soc-platform` VM (10.10.99.20). TheHive automatically creates a new case containing the alert data and observables.
7. **Analyst reviews the case in TheHive**. From the case interface, the analyst can invoke Cortex analyzers against the case observables (IPs, domains, hashes, URLs).
8. **Cortex executes the requested analyzers**, some of which query MISP for threat intelligence context. Results are returned to TheHive and attached to the case.
9. **Analyst uses the enriched context** to decide response actions (containment, escalation, closure).
This flow represents the operational workflow that a SOC L1/L2 analyst executes for each significant alert in a production environment.

