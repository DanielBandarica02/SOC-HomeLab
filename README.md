# SOC-HomeLab

Home Lab SOC built from scratch using Wazuh as EDR for endpoint monitoring 
and Splunk as SIEM for log analysis and threat detection, Snort as IDS, 
and Kali Linux as attack machine against a Windows 11 Pro target. Includes custom 
detection rules, simulated attacks and documented defense strategies. 
Oriented toward Blue Team and SOC analyst roles.

## Architecture
- **Kali Linux** (192.168.10.10) — Attack machine
- **Windows 11 Pro** (192.168.10.20) — Target machine + Wazuh Agent
- **Ubuntu — Wazuh Manager** (192.168.10.30) — EDR backend, alert processing, forwards to Splunk via HEC
- **Ubuntu — Splunk** (192.168.10.40) — SIEM frontend, dashboards, SPL queries, correlation rules

## Data Flow
Windows 11 Pro → Wazuh Agent → Wazuh Manager → Splunk HEC → Splunk SIEM

## Phases
- [X] Phase 1 — Infrastructure setup
- [X] Phase 2 — Wazuh deployment
- [ ] Phase 3 — Splunk deployment + Wazuh integration via HEC
- [ ] Phase 4 — Snort IDS
- [ ] Phase 5 — Detection rules (15+)
- [ ] Phase 6 — Attack simulations and remediation

## Tools Used
Wazuh · Splunk · Snort · Kali Linux · Hydra · Nmap · Metasploit · Burp Suite · John the Ripper · Hashcat
