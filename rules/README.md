# Rules — Production-Ready Detection Code
 
This directory contains the deployable code for all custom detection rules in the SOC HomeLab. Rules are organized by platform and numbered to match their documentation in [`../detection-engineering/rules/`](../detection-engineering/rules/).
 
For the methodology behind each rule, see the corresponding `.md` file in `detection-engineering/rules/`.
 
---
 
## Structure
 
```
rules/
├── wazuh/        XML rules for Wazuh Manager (deploy to /var/ossec/etc/rules/)
└── splunk/       SPL queries for Splunk Saved Searches
```
 
---
 
## Deployment
 
### Wazuh Rules
 
Place XML files in `/var/ossec/etc/rules/local_rules.xml` on the Wazuh Manager, then restart:
 
```bash
sudo /var/ossec/bin/wazuh-control restart
```
 
### Splunk SPL Queries
 
Import each `.spl` file as a Saved Search in Splunk. Recommended schedule: every 5 minutes for correlation-based detections.
 
---
 
## Rule Index
 
| # | Rule | Wazuh | Splunk |
|---|------|-------|--------|
| 1 | SSH Brute Force | [01-ssh-bruteforce.xml](wazuh/01-ssh-bruteforce.xml) | [01-ssh-bruteforce.spl](splunk/01-ssh-bruteforce.spl) |
| 2 | Password Spraying | — | — |
| 3 | RDP Brute Force | — | — |
| 4 | PowerShell Encoded Execution | — | — |
| 5 | Office Spawning Shell | — | — |
| 6-15 | *(to be added)* | — | — |
