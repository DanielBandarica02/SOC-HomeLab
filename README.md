# SOC-HomeLab

Home Lab SOC built from scratch using Wazuh and Splunk as SIEM, Snort as IDS, 
and Kali Linux as attack machine against a Windows 11 Pro target. Includes custom 
detection rules, simulated attacks and documented defense strategies. 
Oriented toward Blue Team and SOC analyst roles.

## Architecture
- **Kali Linux** — Attack machine
- **Windows 11 Pro** — Target machine + Wazuh Agent
- **Ubuntu (Wazuh Manager)** — SIEM backend, alert processing
- **Ubuntu (Splunk)** — SIEM frontend, dashboards and visualization

## Phases
- [X] Phase 1 — Infrastructure setup
- [ ] Phase 2 — Wazuh deployment
- [ ] Phase 3 — Splunk deployment
- [ ] Phase 4 — Snort IDS
- [ ] Phase 5 — Detection rules (15+)
- [ ] Phase 6 — Attack simulations and remediation

## Tools Used
Wazuh · Splunk · Snort · Kali Linux · Hydra · Nmap · Metasploit · Burp Suite · John the Ripper · Hashcat
