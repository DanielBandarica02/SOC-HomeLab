# Scenario 1 — Full Kill Chain against the Dev VLAN
 
## Overview
 
The first attack scenario of the lab exercises the full kill chain against the development environment (VLAN 20): reconnaissance, credential brute force, initial access, host discovery, credential harvesting from local files, lateral movement to a Windows workstation, and two independent persistence layers. Ten distinct phases were executed end-to-end from the Adversary DMZ (Kali on VLAN 66), each phase mapped to a MITRE ATT&CK technique and each phase's telemetry captured in the SIEM.
 
The scenario is deliberately end-to-end. Rather than isolating a single technique, the intent is to demonstrate that the SOC pipeline responds coherently to a **realistic intrusion chain** — one where the attacker moves through the environment in stages, each stage building on the previous. The narrative parallels real incidents: a foothold established via credential attack becomes credential harvesting becomes lateral movement becomes persistence. Every stage produces telemetry, and the SIEM's job is to make each stage visible to the analyst investigating the incident.
 
The scenario was executed against pre-seeded targets: a Linux dev workstation (ws-dev-02) with synthetic credentials placed in developer-typical locations, and a Windows dev workstation (WS-DEV-01) with RDP enabled. This seeding is documented explicitly as environment preparation, not concealed as "found artefacts" — the intent is to model realistic developer workstations, not to fake attacker discoveries.
 
The scenario also surfaces an operational reality that dominates real SOCs: **alert fatigue**. The 10 phases generated 40,034 alerts, of which 87% were duplicates of a single detection rule triggered by the reconnaissance scan. This ratio, and the response to it, is documented at the end of this document as the primary lesson learned from executing the scenario at production-realistic scale.
 
---


## Kill Chain Phases
 
| Phase | Tactic | Actor | Target | 
| ----- | ------ | --------- | ------------- |
| 1 | Reconnaissance | Kali | Subnet 10.10.20.0/24 | 
| 2 | Brute Force SSH |  Kali | ws-dev-02 (10.10.20.20) |
| 3 | Initial Access |  Kali | ws-dev-02 |
| 4 | Discovery post-compromise | Kali (via ws-dev-02) | ws-dev-02 |
| 5 | Credential Access | Kali (via ws-dev-02) | ws-dev-02 |
| 6 | Lateral Movement | ws-dev-02 (pivot) | WS-DEV-01 (10.10.20.10) |
| 7 | Persistence Linux | Kali (via ws-dev-02) | ws-dev-02 |
| 8 | Persistence SSH | Kali (via ws-dev-02) | ws-dev-02 |


---
 
## MITRE ATT&CK coverage
 
| Phase | Tactic | Technique | Sub-technique |
| ----- | ------ | --------- | ------------- |
| 1 | Reconnaissance | T1046 — Network Service Discovery | — |
| 2 | Credential Access | T1110 — Brute Force | T1110.001 — Password Guessing |
| 3 | Initial Access | T1078 — Valid Accounts | — |
| 4 | Discovery | T1082 — System Information Discovery | — |
| 4 | Discovery | T1087 — Account Discovery | — |
| 4 | Discovery | T1057 — Process Discovery | — |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.001 — Credentials In Files |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.003 — Bash History |
| 5 | Credential Access | T1552 — Unsecured Credentials | T1552.004 — Private Keys |
| 6 | Lateral Movement | T1021 — Remote Services | T1021.001 — Remote Desktop Protocol |
| 6 | Command and Control | T1572 — Protocol Tunneling | — |
| 7 | Persistence | T1053 — Scheduled Task/Job | T1053.003 — Cron |
| 8 | Persistence | T1098 — Account Manipulation | T1098.004 — SSH Authorized Keys |

---
 
## Environment preparation
 
Before execution, both target hosts were seeded with synthetic credentials and configuration that model a realistic developer environment. This seeding is documented explicitly rather than presented as attacker "discoveries" of pre-existing state — the scenario is a controlled lab exercise, and the seeded data represents what a real developer workstation typically contains

### ws-dev-02 seeding
 
Local user `arodriguez` was set with the password `summer2024`, deliberately chosen as a common password appearing in the first pages of the `rockyou.txt` wordlist. This models the reality that developer accounts frequently use weak passwords when the environment is treated as "internal" and low-risk.
 
Two artefacts were placed in the user's home directory to model credential-hoarding patterns commonly found on developer workstations:
 
**~/.env** — A file simulating environment variables for a local development stack:

```
DB_HOST=10.10.20.10
DB_USER=devadmin
DB_PASSWORD=DevPassw0rd2024!
API_KEY=sk_test_abcdef1234567890
```
 
**~/notes.txt** — A plain-text notes file simulating an ad-hoc password reminder:

```
Reminders:
- WS-DEV-01 login: ¨kevin hernandez¨ / Kevin2026!
- Wifi office: SoclabWifi / SocLab2026!
- VPN backup: dbandarica / DAniel2026!
```


The key fingerprint is: SHA256:/HQT9NG+KnLFqJz5tiTb6sTgK/FLL7hZwxe5al337eQ

### WS-DEV-01 preparation
 
The user `kevin hernandez` was assigned the password `Kevin2026!` — the same password referenced in the ws-dev-02 notes file, creating a credential chain that would allow lateral movement once the notes were discovered.

### Snapshots
 
Both hosts were snapshotted post-preparation as `scenario-1-ready`, enabling rapid reset after the scenario for future re-execution or variation.

---

# Phase 1 — Reconnaissance
 
**Objective:** Enumerate accessible hosts and services in VLAN 20 from the adversary DMZ.
 
**Tools used:** nmap
 
**MITRE mapping:** T1046 — Network Service Discovery
 
### Execution
 
An initial ping sweep was attempted to identify live hosts:

```bash
nmap -sn 10.10.20.0/24
```
 
As expected, this returned no hosts up. The pfSense firewall rules from Phase 3 do not permit ICMP from VLAN 66 to VLAN 20, and the ping probes were silently dropped. Every dropped packet, however, generated a pfSense filterlog event forwarded to Wazuh via syslog.

The attack pivoted to a TCP SYN scan against common service ports, using `-Pn` to skip the ICMP host discovery phase:
 
```bash
sudo nmap -Pn -sS -p 22,3389,445,80,443,3306 10.10.20.0/24
```

Two live hosts were identified:

![Nmap Hosts Identified](../../screenshots/04-attack/01-kill-chain/01-nmap-scan-identified-1.png)

![Nmap Hosts Identified](../../screenshots/04-attack/01-kill-chain/02-nmap-scan-identified-2.png)


A follow-up service enumeration was performed against the two discovered hosts:
 
```bash
sudo nmap -Pn -sV -sC -p 22,3389 10.10.20.20 10.10.20.10 -oN /tmp/recon-vlan20.txt
```

![Nmap Hosts Identified](../../screenshots/04-attack/01-kill-chain/03-nmap-scan-identified-3.png)

The output revealed OpenSSH 9.6p1 on ws-dev-02 and the Windows RDP service on WS-DEV-01. The Linux SSH banner revealed the version, providing the attacker with information useful for identifying known vulnerabilities.

### Detection

The reconnaissance activity was the highest-volume detection event of the entire scenario. The pfSense firewall rules configured in Phase 3 dropped every packet from VLAN 66 that did not match the explicitly permitted paths (SSH and RDP to VLAN 20 and VLAN 10). The `/24` scan across six ports produced approximately 34,800 individual pfSense filterlog events, each processed by the custom `pfsense-custom-header` decoder and forwarded as an alert.

![Wazuh Events Reconnaissance](../../screenshots/04-attack/01-kill-chain/04-wazuh-logs-reconnaissance-1.png)

These events were triggered after the initial two Nmap scans.

![Wazuh Events Reconnaissance](../../screenshots/04-attack/01-kill-chain/05-wazuh-logs-reconnaissance-2.png)

These events were triggered after the service enumeration scan.

![Wazuh Events](../../screenshots/04-attack/01-kill-chain/06-wazuh-logs.png)

The volume of these alerts represents a real SOC challenge that will be analysed in the Alert Volume Analysis section at the end of this document.

---
 
## Phase 2 — Credential brute force
 
**Objective:** Obtain valid credentials for the discovered SSH service.
 
**Tools used:** hydra with `rockyou.txt`
 
**MITRE mapping:** T1110.001 — Password Guessing
 
### Execution
 
A wordlist subset was prepared to keep the brute force bounded within the demo timeframe:
 
```bash
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
head -n 1000 /usr/share/wordlists/rockyou.txt > /home/wordlist_small.txt
```
 
The `arodriguez` username was chosen as the target based on prior enumeration of the Dev environment context. The brute force ran with four parallel threads and verbose output:

![Hydra Brute Force](../../screenshots/04-attack/01-kill-chain/07-hydra-attempts.png)

The `-f` flag terminates the attack on the first successful match, minimising noise. The password was cracked after approximately 240 seconds of attempts:

![Hydra Brute Force Completed](../../screenshots/04-attack/01-kill-chain/08-hydra-completed.png)

### Detection
 
The brute force generated three distinct alert categories in Wazuh, each corresponding to a different decoder layer processing the authentication events:

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/06-wazuh-logs.png)

| Alert type | Approximate count | Wazuh decoder |
| ---------- | ----------------- | ------------- |
| `syslog: User authentication failure` | 2,300 | syslog (auth.log) |
| `syslog: User authentication failure` (secondary rule) | 390 | syslog (auth.log) |
| `PAM: User login failed` | 337 | pam (auth.log via journald) |

The multiple counts reflect Wazuh's layered decoding — the same underlying event (a failed SSH login) is captured by the syslog decoder reading `/var/log/auth.log` and by the PAM decoder recognising the specific PAM failure signature. Both produce alerts, providing defence-in-depth against decoder-specific detection gaps.

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/09-wazuh-logs-brute-force.png)

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/10-wazuh-logs-brute-force-2.png)

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/11-wazuh-logs-brute-force-3.png)




