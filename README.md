# SOC-HomeLab

Home lab SOC built from scratch using Wazuh as EDR for endpoint monitoring and Splunk as SIEM for log analysis and threat detection, Snort as IDS, and Kali Linux as attack machine against an Ubuntu Desktop target. Includes custom detection rules, simulated attacks, and documented defense strategies. Oriented toward Blue Team and SOC analyst roles.

## Architecture

- **Kali Linux** (192.168.10.10) — Attack machine
- **Ubuntu Desktop 24.04** (192.168.10.20) — Target machine + Wazuh Agent
- **Ubuntu — Wazuh Manager** (192.168.10.30) — EDR backend, alert processing, forwards to Splunk via HEC
- **Ubuntu — Splunk** (192.168.10.40) — SIEM frontend, dashboards, SPL queries, correlation rules

## Data Flow

```
Ubuntu Desktop → Wazuh Agent → Wazuh Manager → Splunk HEC → Splunk SIEM
```

## Phases

- [x] Phase 1 — Infrastructure setup
- [x] Phase 2 — Wazuh deployment
- [X] Phase 3 — Splunk deployment + Wazuh integration via HEC
- [ ] Phase 4 — Snort IDS
- [ ] Phase 5 — Detection rules (15+)
- [ ] Phase 6 — Attack simulations and remediation

## Tools Used

Wazuh · Splunk · Snort · Kali Linux · Hydra · Nmap · Metasploit · Burp Suite · John the Ripper · Hashcat

## Skills Demonstrated

- SIEM configuration and log ingestion pipeline
- EDR deployment and endpoint agent management
- Network segmentation and isolated lab design
- Threat detection rule writing (Wazuh + Splunk SPL)
- Attack simulation and blue team response documentation
- MITRE ATT&CK framework mapping
