# Rule 100020: Discovery Command Sequence Detection
 
## Metadata
| Field | Value |
|-------|-------|
| Rule ID | `100021` |
| Severity | Medium |
| MITRE ATT&CK Tactic | Discovery |
| MITRE ATT&CK Technique | T1082 — System Information Discovery / T1087.001 — Account Discovery: Local Account / T1057 — Process Discovery / T1049 — System Network Connections Discovery |
| Data Source | auditd (via Wazuh Linux agent) |
| Platform | Linux |
| Status | Active |
 
---
 
## Threat Context
 
### Description
Fires when a single user account executes six or more distinct reconnaissance commands within a 60-second window. Individual discovery commands (`whoami`, `id`, `ps`, `ss`, `ip a`, `uname -a`, and similar) are legitimate operations that appear in normal system administration. The rule detects the pattern of clustered execution — the signature of an attacker enumerating a newly compromised host to understand its role, users, and network position before pivoting.
 
### Real-World Usage
Post-compromise host enumeration is a universal attacker behaviour, present in virtually every recorded intrusion. Threat intelligence catalogues consistently identify the same command patterns across independent operators: MITRE ATT&CK's data on T1082 and T1057 cites usage by APT41, Sandworm, Turla, Winnti Group, and multiple ransomware families (LockBit, ALPHV, Play). Public post-breach reports from Mandiant, CrowdStrike, and Elastic Security Labs describe the pattern as "living-off-the-land discovery" — attackers deliberately use built-in Linux tools rather than deploying custom binaries, precisely to blend into normal administrative activity and evade signature-based detection.
 
### Why This Matters
Discovery is the phase between initial access and impact. It is the window during which the attacker maps the compromised environment and decides how to proceed. Detecting discovery gives the SOC an opportunity to intervene before lateral movement, credential harvesting, or data exfiltration begins — often the difference between contained incident and full breach. This rule catches the burst pattern that distinguishes attacker reconnaissance from normal administrative usage, providing early warning that a foothold has become active exploration.
 
---
 
## Detection Strategy
 
### Logic
The detection uses auditd `execve` syscall watches on eleven common Linux reconnaissance commands, all sharing a single key (`discovery_command`). A Wazuh custom rule then applies frequency-based aggregation: when 6 or more matching events occur from the same user ID (`audit.uid`) within 60 seconds, a Medium-severity alert is generated with mappings to four MITRE Discovery techniques.
 
The design deliberately combines multiple ATT&CK techniques under a single rule rather than creating one rule per command. This is because attacker reconnaissance is characterised by breadth — probing system info, users, processes, and network state — not by any single command in isolation. A rule per command would generate constant noise from individual sysadmin usage; the aggregation captures the meaningful pattern.
 
### Data Source Requirements
- Source: auditd via Wazuh Linux agent
- Required fields: `audit.key`, `audit.exe`, `audit.uid`
- Prerequisites:
  - auditd installed and running
  - execve watches deployed in `/etc/audit/rules.d/discovery-detection.rules` 
  - Wazuh Linux agent configured with audit log ingestion
  - Wazuh built-in ruleset group `audit` operational
    
### Thresholds
- **frequency = 6** — chosen as the smallest count that reliably distinguishes attacker enumeration from ad-hoc admin usage. Legitimate administrators typically execute 1-3 discovery commands during focused troubleshooting; six or more clustered together signals systematic enumeration.
- **timeframe = 60 seconds** — a 60-second window captures the fast, sequential execution characteristic of attacker scripting while giving legitimate admins headroom for slower, thoughtful command sequences interrupted by output review.
- **Level 8 (Medium)** — deliberately below the Critical severity of Persistence and Credential Access rules. Discovery is high-signal but individually less operationally urgent; the appropriate response is investigation, not immediate containment. The Medium severity keeps the alert prominent in dashboards without triggering the escalation protocols reserved for confirmed compromise.
### Empirical calibration note
This rule carries the highest false positive risk in the ruleset. During threshold selection, values of `frequency=4` were considered but rejected because typical troubleshooting sessions (checking a failing service) often generate 4-5 discovery commands legitimately. Values of `frequency=8` were also considered but rejected because slower attackers using tab completion or reviewing output between commands could stay below the threshold. The chosen value of 6 was empirically validated against both attacker patterns (Scenario 1 reconnaissance phase) and legitimate admin patterns (routine troubleshooting sessions on the lab hosts).
 
---
 
## Implementation
 
### Prerequisite — auditd execve watches
 
```bash
/etc/audit/rules.d/discovery-detection.rules

# Watch execution of common discovery commands
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/whoami -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/id -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/w -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/last -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/ps -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/ss -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/netstat -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/ip -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/uname -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/hostname -k discovery_command
-a always,exit -F arch=b64 -S execve -F path=/usr/bin/crontab -k discovery_command
```
 
Notes on the watch configuration:
- Unlike file watches (`-w`), execve watches use `-a always,exit -S execve` syntax to capture syscall events at process exit.
- `-F arch=b64` restricts to 64-bit executions, matching modern Ubuntu installations. In dual-arch environments, a paired `arch=b32` block would be added.
- `-F path=/usr/bin/X` restricts by absolute path. This means attackers who copy the binary to another location or use a full path variant would evade detection; on this trade-off, see the False Positives section.
- All watches share the single key `discovery_command`. This design choice enables the aggregation rule to fire based on total discovery activity regardless of which specific commands were used.
- The eleven commands cover the four ATT&CK techniques mapped in the rule: system info (`uname`, `hostname`), account discovery (`whoami`, `id`, `w`, `last`), process discovery (`ps`), and network state (`ss`, `netstat`, `ip`), plus scheduled tasks (`crontab`).
### Wazuh Rule (XML)
 
```xml
<group name="audit,custom,">
  <rule id="100020" level="8" frequency="6" timeframe="60">
    <if_matched_group>audit</if_matched_group>
    <field name="audit.key">discovery_command</field>
    <same_field>audit.uid</same_field>
    <description>Discovery command sequence detected - user uid $(audit.uid) executed 6+ reconnaissance commands in 60 seconds</description>
    <mitre>
      <id>T1082</id>
      <id>T1087.001</id>
      <id>T1057</id>
      <id>T1049</id>
    </mitre>
    <group>attack,discovery,recon,</group>
  </rule>
</group>
```
 
Key structural decisions:
- `<if_matched_group>audit</if_matched_group>` scopes matching to Wazuh-processed audit events, ensuring the fields are populated.
- `<field name="audit.key">discovery_command</field>` filters to events tagged by the execve watches.
- `<same_field>audit.uid</same_field>` groups by the executing user, not by source IP. Discovery occurs on-host after the attacker has established foothold; grouping by UID correctly attributes the pattern to the compromised account.
---
 
## Atomic Testing
 
### Test Command
From an SSH session on the target host, execute seven common discovery commands in sequence:
 
```bash
whoami
id
uname -a
last -n 5
ps aux | head -5
ss -tulpn
ip a
```
 
The commands complete in approximately 15-20 seconds, exceeding the frequency threshold with margin.
 
### Expected Result
One alert in `wazuh-alerts-*` with:
- `rule.id: 100021`
- `rule.level: 8`
- `rule.description` containing "Discovery command sequence detected - user uid 1000 executed 6+ reconnaissance commands in 60 seconds"
- `rule.mitre.id: ["T1082", "T1087.001", "T1057", "T1049"]`
- `rule.mitre.tactic: ["Discovery"]`
- `data.audit.uid: 1000` (or whichever UID executed)
In parallel, individual audit events for each command execution remain available in the archives for forensic drill-down.
 
### Validation Screenshot
![Rule 100021 validation](../../screenshots/05-detection-rules/11-rule-100020-discovery-command-sequence-detection.png)
 
---
 
## False Positives
 
### Known FP Scenarios
- Administrators performing focused troubleshooting sessions frequently execute burst sequences of discovery commands (checking user, network state, and process list when diagnosing a failing service).
- Automation scripts that gather host inventory data run collections of discovery commands during scheduled inventory sweeps.
- Configuration management agents (Ansible facts collection, Puppet facter) execute discovery commands as part of every run to build system facts.
- Shell startup files with heavy customisation may execute several discovery commands automatically on login, potentially crossing the threshold for interactive sessions.
- Debugging tools and diagnostic scripts (sosreport, sysdiagnose) execute large numbers of discovery commands as part of their data collection.
  
### Mitigations
- The threshold of 6 commands in 60 seconds was chosen to sit above typical single-purpose troubleshooting; environments with heavy admin activity may need to raise the threshold to 8 or 10.
- Known automation source UIDs (Ansible, Puppet service accounts) should be excluded via `<field name="audit.uid" negate="yes">998|999</field>` referencing the specific automation UIDs in the target environment.
- Discovery execution from the `root` account (uid 0) during known maintenance windows can be de-prioritised in triage; discovery from non-privileged UIDs during off-hours warrants immediate attention.
- Attackers who use full paths (`/bin/whoami` instead of `whoami`) or copied binaries to non-standard locations will evade the path-specific watches. Comprehensive coverage would require watching by inode or by process name, at the cost of higher false positive rates. The current approach accepts this evasion trade-off in favour of precision.
- In production, this rule benefits significantly from behavioural baselining: users whose normal activity includes clustered discovery commands should be profiled and the rule tuned per-user or per-role.
---
 
## References
- [MITRE ATT&CK T1082 — System Information Discovery](https://attack.mitre.org/techniques/T1082/)
- [MITRE ATT&CK T1087.001 — Account Discovery: Local Account](https://attack.mitre.org/techniques/T1087/001/)
- [MITRE ATT&CK T1057 — Process Discovery](https://attack.mitre.org/techniques/T1057/)
- [MITRE ATT&CK T1049 — System Network Connections Discovery](https://attack.mitre.org/techniques/T1049/)
- [Linux auditd documentation — Syscall audit rules](https://man7.org/linux/man-pages/man8/auditctl.8.html)
- [MITRE ATT&CK Discovery tactic overview](https://attack.mitre.org/tactics/TA0007/)
- Internal reference: `docs/04-attack-scenarios/01-full-kill-chain-vlan-dev.md` (Phase 4 — the discovery sequence this rule detects)
