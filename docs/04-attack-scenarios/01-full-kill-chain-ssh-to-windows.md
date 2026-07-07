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
 
### Objective

Enumerate the VLAN 20 subnet from Kali to identify reachable hosts and open services. This is the first activity a post-perimeter attacker performs, establishing the internal attack surface.
 
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
 
### Objective

Obtain valid credentials for SSH access to ws-dev-02 by dictionary attack against the known user `arodriguez`.
 
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

- **2,321 syslog authentication failure events** — SSH's own logging of each failed attempt
- **390 syslog user authentication failure events** — supplementary auth log entries
- **337 PAM login failure events** — the PAM subsystem's independent record of each failure

The correct password attempt was logged as `authentication_success`. Combined with the 3,027 failure events from the same source IP within 45 seconds, this pattern would trigger an obvious "brute force detected" analysis in a mature SOC. Wazuh's built-in ruleset detected each individual failure but did not currently aggregate them into a single high-severity alert, a gap identified for detection engineering improvement.

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/09-wazuh-logs-brute-force.png)

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/10-wazuh-logs-brute-force-2.png)

![Hydra Brute Force Detection](../../screenshots/04-attack/01-kill-chain/11-wazuh-logs-brute-force-3.png)

---
 
## Phase 3 — Initial access
 
### Objective

Establish an interactive SSH session on ws-dev-02 using the credentials recovered in Phase 2. This transitions the attacker from "external observer" to "local user on internal host".
 
**Tools used:** OpenSSH client
 
**MITRE mapping:** T1078 — Valid Accounts
 
### Execution
 
With valid credentials in hand, direct SSH access was established:
 
![SSH Initial Access](../../screenshots/04-attack/01-kill-chain/12-initial-access-ssh.png)
 
The login succeeded. The prompt changed to `arodriguez@ws-dev-02:~$`, indicating shell access.
 
### Detection
 
A single `authentication_success` event was generated, with `srcip: 10.10.66.10`. In isolation, this event carries level 3 severity (informational). Its criticality is context-dependent — a successful login from Kali's IP address to a dev workstation, immediately following 3,027 authentication failures from the same source, is highly indicative of a successful brute-force compromise. This kind of temporal correlation is exactly what a mature SOC ruleset would flag as a compound alert.

![SSH Initial Access Detection](../../screenshots/04-attack/01-kill-chain/13-initial-access-ssh-detection.png)

---
 
## Phase 4 — Discovery (T1082, T1087.001, T1057, T1049)
 
### Objective

Enumerate the compromised host to understand the environment, identify additional targets, and locate credentials or elevation opportunities.
 
### Execution
 
The following commands were executed from the SSH session:
 
```bash
uname -a                           # kernel version, host name
lsb_release -a                     # OS distribution
whoami                             # current user
id                                 # UID, GID, groups
cat /etc/passwd | grep -v nologin  # human users on the system
last -n 20                         # recent login history
w                                  # currently logged-in users
ps auxf                            # process tree
ss -tulpn                          # listening services
ip a                               # network interfaces
sudo -l                            # sudo privileges for arodriguez
crontab -l                         # user's cron entries
ls -la /etc/cron*                  # system cron directories
```
 
Key findings:
- Host: `ws-dev-02` running Ubuntu 24.04.1 LTS
- User `arodriguez` (UID 1001), member of `sudo` group
- `sudo -l` revealed `(ALL : ALL) ALL` — full sudo privileges
- Additional local user `¨kevin hernandez¨` visible in `/etc/passwd` (though not the RDP target — that's on WS-DEV-01)
- No unusual services listening beyond sshd
- No suspicious cron entries yet (persistence would be added in Phase 7)

### Wazuh detection
 
The discovery commands were logged by auditd to the extent the audit rules covered them. Commands executed via bash shell generated syscall events indirectly through auditd's process monitoring, though the Wazuh built-in ruleset does not fire prominent alerts for individual discovery commands — this is by design, as flagging every `whoami` or `ps` would produce untenable false-positive rates in normal operation.
 
Two notable exceptions did fire:
- `sudo -l` produced a "sudo command executed" event
- The `w` command's read of `/var/run/utmp` was captured by auditd if watch rules on that path were configured

---
 
## Phase 5 — Credential Access (T1552.001, T1552.003, T1552.004)
 
### Objective
 
Harvest credentials from the compromised host's filesystem. This phase leverages the seeded artefacts from Environment Preparation.
 
### Execution
 
**Search for common credential-containing files:**
```bash
find /home -type f -name "*.env" 2>/dev/null
find /home -type f -name "*password*" 2>/dev/null
find /home -type f -name "*.key" 2>/dev/null
find /home -type f -name "notes*" 2>/dev/null
```
 
**Read the discovered files:**
```bash
cat ~/.env                 # database credentials, API keys
cat ~/notes.txt            # plaintext credentials for multiple systems
cat ~/.ssh/service_key     # private SSH key
cat ~/.bash_history         # commands with embedded passwords
```
 
**Grep for password references in system paths:**
```bash
sudo grep -r "password" /var/log 2>/dev/null | head -20
```
 
Credentials harvested from the host:
- **DB access:** `devadmin` / `DevPassw0rd2024!` (from `.env`)
- **API key:** `sk_test_abcdef1234567890` (from `.env`)
- **WS-DEV-01 RDP:** `kevin hernandez` / `Kevin2026!` (from `notes.txt`) — this is the key credential enabling lateral movement in Phase 6
- **VPN backup:** `dbandarica` / `DAniel2026!` (from `notes.txt`)
- **SSH service key** for automated authentication

### Wazuh detection
 
The credential access phase generated the least telemetry of any active phase. Reading files with `cat` produces syscall events but is not flagged by default rules — a legitimate user reads their own files continuously during normal work. Detection engineering opportunity: an auditd watch on `~/.env`, `~/.ssh/`, and `~/.bash_history` reads would catch this pattern with acceptable false-positive rate on production servers where these files rarely change or are accessed programmatically.



