# Rule 1: SSH Brute Force Detection
 
## Metadata
 
| Field | Value |
|-------|-------|
| Rule ID | `100001` (Wazuh) |
| Severity | High (Level 12) |
| MITRE ATT&CK Tactic | Credential Access |
| MITRE ATT&CK Technique | [T1110.001 — Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) |
| Data Source | Wazuh (sshd logs from `/var/log/auth.log`) |
| Platform | Linux |
| Status | Active |
 
---
 
## Threat Context
 
### Description
Detects SSH brute force attacks against Linux endpoints by correlating multiple authentication failures from the same source IP within a short time window. The rule fires when 10 or more failed SSH authentication attempts originate from the same source IP in a 5-minute window, regardless of the target user.
 
### Real-World Usage
SSH brute force is the most common initial access vector against internet-exposed Linux infrastructure. It is used by:
 
- **Mass-scanning botnets** (Mirai, Mozi, FritzFrog) targeting any exposed SSH service
- **Ransomware affiliates** (Conti, BlackCat) during the initial access phase against Linux servers
- **Cryptojacking groups** (TeamTNT, Outlaw) compromising Linux servers for cryptocurrency mining
- **APT actors** in opportunistic operations when no zero-day is available
### Why This Matters
If successful, the attacker obtains an interactive shell on the target system, enabling:
1. Privilege escalation via local exploits or `sudo` misconfigurations
2. Establishment of persistence (`~/.ssh/authorized_keys`, cron jobs, systemd units)
3. Lateral movement to other hosts on the network
4. Credential reuse to access management infrastructure (Wazuh, Splunk, etc.)
SSH is the **#1 initial access vector** for Linux compromise according to multiple incident response reports.
 
---
 
## Detection Strategy
 
### Logic
When a single source IP generates 10 or more SSH authentication failures (Wazuh rule 5760) against a single target host within a 5-minute window, an alert is generated. The Splunk SPL companion query enriches the alert with the number of unique users targeted, attack rate (attempts per minute), and a calculated severity level based on volume.
 
### Data Source Requirements
- **Source:** `index=wazuh`
- **Required base rule:** Wazuh `5760` (`sshd: authentication failed`)
- **Required fields:**
  - `data.srcip` — attacker source IP
  - `data.dstuser` — target user account
  - `agent.name` — victim hostname
  - `_time` — event timestamp
- **Prerequisites:** SSH password authentication enabled on the target; `auth.log` ingestion via Wazuh agent
### Thresholds
 
| Parameter | Value | Justification |
|-----------|-------|---------------|
| `frequency` | 10 | A legitimate user does not fail 10+ authentications in 5 minutes. Higher than Wazuh's native threshold (8) to reduce false positives when grouping only by source IP. |
| `timeframe` | 300 seconds | Automated brute force generates hundreds of events in this range. Low-and-slow attacks are out of scope and handled by separate detection logic. |
| `level` | 12 | High severity — indicates an active attacker with clear intent. Justifies immediate SOC response. |
 
---
 
## Implementation
 
### Wazuh Rule (XML)
 
File location: `/var/ossec/etc/rules/local_rules.xml`
 
```xml
<group name="custom,authentication_failed,ssh,brute_force,">
 
  <rule id="100001" level="12" frequency="10" timeframe="300">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip />
    <description>Custom: SSH brute force — 10+ failed SSH authentications from same source IP within 5 minutes</description>
    <mitre>
      <id>T1110.001</id>
    </mitre>
    <group>authentication_failures,ssh,attack,</group>
  </rule>
 
</group>
```
 
**Key elements:**
- `<if_matched_sid>5760</if_matched_sid>` — depends on the base SSH failure rule
- `<same_source_ip />` — correlates events by source IP automatically
- `<frequency>10</frequency>` + `<timeframe>300</timeframe>` — the correlation logic
- `<mitre>` block — automatic mapping for Wazuh dashboard's MITRE ATT&CK view
### Splunk SPL (Enrichment Query)
 
Designed to be saved as a scheduled Splunk Saved Search running every 5 minutes:
 
```spl
index=wazuh rule.id=5760 earliest=-5m@m latest=now
| stats
    count                            AS failed_attempts,
    dc(data.dstuser)                 AS unique_users_targeted,
    values(data.dstuser)             AS users_list,
    min(_time)                       AS first_attempt_epoch,
    max(_time)                       AS last_attempt_epoch
  BY data.srcip, agent.name
| eval attack_duration_sec = last_attempt_epoch - first_attempt_epoch
| eval attempts_per_minute = if(attack_duration_sec > 0,
                                round(failed_attempts / (attack_duration_sec/60), 2),
                                failed_attempts)
| eval severity = case(
    failed_attempts >= 100, "Critical",
    failed_attempts >= 50,  "High",
    failed_attempts >= 10,  "Medium",
    true(),                 "Low"
  )
| where failed_attempts >= 10
| eval first_attempt = strftime(first_attempt_epoch, "%Y-%m-%d %H:%M:%S"),
       last_attempt  = strftime(last_attempt_epoch,  "%Y-%m-%d %H:%M:%S")
| sort - failed_attempts
| table data.srcip, agent.name, failed_attempts, unique_users_targeted,
        attempts_per_minute, severity, first_attempt, last_attempt, users_list
```
 
**Why both implementations:**
- The Wazuh rule generates the atomic alert in near real-time (latency ~1-5s) and feeds the SIEM
- The SPL query provides the enriched analyst view, surfacing attack rate, target users, and dynamic severity in a single pane
---
 
## Atomic Testing
 
### Test Command
Executed from the Kali Linux attacker (192.168.10.10) against the Ubuntu Desktop target (192.168.10.20):
 
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.10.20 -t 4 -V
```
 
| Flag | Function |
|------|----------|
| `-l root` | Fixed target user |
| `-P rockyou.txt` | Password wordlist |
| `-t 4` | 4 parallel threads |
| `-V` | Verbose output per attempt |
 
Run for 60-90 seconds, then terminate with `Ctrl+C`. Generates approximately 200-300 failed authentication attempts.
 
### Expected Result
The Wazuh custom rule `100001` fires multiple times (one alert per block of 10 correlated failures). The Splunk enrichment query returns a row like this:

| data.srcip | agent.name | failed_attempts | unique_users_targeted | attempts_per_minute | severity |
|------------|------------|-----------------|----------------------|---------------------|----------|
| 192.168.10.10 | ubuntu-target | 190 | 1 (root) | 69.09 | Critical |


### Validation Commands
 
In the Wazuh Manager:
```bash
sudo grep -c "Rule: 100001" /var/ossec/logs/alerts/alerts.log
```
 
In Splunk:
```spl
index=wazuh rule.id=100001 earliest=-15m
| table _time, data.srcip, agent.name, rule.description, rule.level
```
 
### Validation Screenshots

- ![Enriched SPL Query Result](/screenshots/phase6/01-ssh-bruteforce-splunk-table.png) — Enriched SPL query result
  
---
 
## False Positives
 
### Known FP Scenarios
 
| Scenario | Likelihood | Impact |
|----------|------------|--------|
| Script with stale credentials (rsync, ansible, backup jobs) retrying authentication | Medium | Could generate periodic FPs from internal IPs |
| Authorized penetration test | Low | Will trigger but expected during engagement |
| User with broken keyboard or password manager issues retrying manually | Low | Threshold of 10 attempts in 5 minutes usually filters this out |
| Misconfigured monitoring tool checking SSH availability | Medium | Could generate FPs if probes fail authentication |
 
### Mitigations
 
1. **Whitelist trusted internal IPs** in the Wazuh rule with a follow-up exclusion rule:
   ```xml
   <rule id="100002" level="0">
     <if_sid>100001</if_sid>
     <srcip>192.168.10.50</srcip>  <!-- example: trusted automation host -->
     <description>SSH brute force from whitelisted source - suppressed</description>
   </rule>
   ```
 
2. **Coordinate with pentest team** to temporarily whitelist their source IPs in Splunk via lookup table or supresión.
3. **Tune threshold per host** if specific systems have legitimate high-retry patterns (rare but documented).
---
 
## Detection-Prevention Tradeoff
 
| Aspect | Status |
|--------|--------|
| **Detection** | This rule (100001) — alerts on brute force in real-time |
| **Prevention** | Disable `PasswordAuthentication` in `/etc/ssh/sshd_config`; enforce key-based authentication only |
| **Why prevention is not always feasible** | Some legacy systems require password fallback, emergency access scenarios, or scripts that cannot be migrated to key-based auth |
| **Recommended layered approach** | `fail2ban` + MFA + bastion host (restricted SSH ingress) + this detection rule |
 
In production environments, this rule should be one layer of defense — not the only one.
 
---
 
## References
 
- [MITRE ATT&CK T1110.001 — Password Guessing](https://attack.mitre.org/techniques/T1110/001/)
- [Wazuh Documentation — Custom Rules](https://documentation.wazuh.com/current/user-manual/ruleset/custom.html)
- [Atomic Red Team T1110.001](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1110.001/T1110.001.md)
- [CISA AA20-302A — Ransomware activity targeting K-12 sector](https://www.cisa.gov/news-events/cybersecurity-advisories/aa20-302a) — references brute force as initial access
---
 
