# Rule 2: Password Spraying Detection

## Metadata

| Field | Value |
|-------|-------|
| Rule ID | `100002` (Wazuh) |
| Severity | Critical (Level 13) |
| MITRE ATT&CK Tactic | Credential Access |
| MITRE ATT&CK Technique | [T1110.003 — Brute Force: Password Spraying](https://attack.mitre.org/techniques/T1110/003/) |
| Data Source | Wazuh (Windows Security Event Log via `eventchannel`) |
| Platform | Windows / Active Directory |
| Status | Active |

---

## Threat Context

### Description
Detects password spraying attacks against the Active Directory domain `lab.local`. Unlike traditional brute force, password spraying inverts the pattern: a single common password is tested against many distinct user accounts, deliberately staying below account lockout thresholds. The Wazuh rule fires when 5 or more failed authentication events correlate from the same source IP within a 30-minute window. The Splunk SPL companion query performs the statistical classification to distinguish password spraying from brute force based on the `attempts-per-user` ratio.

### Real-World Usage
Password spraying has become one of the most common Initial Access vectors against Active Directory environments precisely because it evades account lockout policies that defeat traditional brute force. Documented use by:

- **APT29 (Midnight Blizzard / Cozy Bear)** — 2024 compromise of Microsoft executive accounts via password spraying against a legacy account without MFA
- **APT33 (Refined Kitten)** — operations against energy sector targets
- **Ransomware affiliates** — common initial access vector for groups like LockBit, BlackCat, and Akira
- **Initial Access Brokers** — sell access obtained through spraying on criminal forums

### Reference Incidents
- **CISA AA24-057A** — alert on Russian state-sponsored actors using password spraying against critical infrastructure
- **Microsoft 2024 incident** — APT29 obtained access to Microsoft senior leadership emails via spraying a non-MFA legacy account
- **Okta 2023 breach** — included password spraying as part of the attack chain

### Why This Matters
A single successful spray gives the attacker **valid domain credentials**, which bypass many downstream detections because the resulting authentication appears legitimate. The compromised account becomes the foothold for lateral movement, privilege escalation, and persistence. Detection at the spraying stage — before successful authentication — is therefore critical.

---

## Detection Strategy

### Logic
The signature pattern of password spraying differs fundamentally from brute force:

| Metric | Brute Force | Password Spraying |
|--------|-------------|-------------------|
| Total failures | High (100s) | Medium (10-50) |
| Unique users targeted | Low (1-3) | **High (5+)** |
| Attempts per user (ratio) | High (>10) | **Low (1-3)** |

The Wazuh rule fires on the baseline temporal correlation. The Splunk SPL applies the statistical classification using `attempts-per-user` to determine whether the pattern is spraying, brute force, or credential stuffing.

### Data Source Requirements
- **Source:** `index=wazuh`
- **Required base rule:** Wazuh `60122` (`Logon Failure - Unknown user or bad password`)
- **Required event channel:** `Security` (must be forwarded by Wazuh Agent on the Domain Controller via `agent.conf`)
- **Required fields:**
  - `data.win.eventdata.ipAddress` — attacker source IP (note: Wazuh decoder uses `ipAddress` for Windows events, not `srcip`)
  - `data.win.eventdata.targetUserName` — user being authenticated against
  - `data.win.eventdata.subStatus` — failure reason code (`0xC000006A` = bad password)
  - `data.win.eventdata.logonType` — type of logon attempted (3 = network, typical for remote spraying)
  - `agent.name` — victim hostname (Domain Controller)
- **Prerequisites:**
  - Windows audit policy configured to log Account Logon failures (Event ID 4625)
  - Wazuh shared `agent.conf` includes the `Security` event channel:
    ```xml
    <agent_config os="Windows">
      <localfile>
        <location>Security</location>
        <log_format>eventchannel</log_format>
      </localfile>
    </agent_config>
    ```

### Thresholds

| Parameter | Value | Justification |
|-----------|-------|---------------|
| `frequency` | 5 | A legitimate user does not target multiple accounts. Five failed logons from a single IP within the time window is statistically anomalous. |
| `timeframe` | 1800 seconds (30 min) | Attackers space attempts to evade account lockout policies. A wider window than brute force, but still indicative of automated activity. |
| `level` | 13 | Critical — password spraying often succeeds because it tests common/known passwords against many accounts. Justifies immediate SOC response and investigation of subsequent successful logons from the same source. |

---

## Implementation

### Wazuh Rule (XML)

File location: `/var/ossec/etc/rules/local_rules.xml`

```xml
<group name="custom,authentication_failed,windows,password_spraying,">

  <rule id="100002" level="13" frequency="5" timeframe="1800">
    <if_matched_sid>60122</if_matched_sid>
    <same_field>win.eventdata.ipAddress</same_field>
    <description>Custom: Password spraying detected - 5+ failed logon attempts from same source IP within 30 minutes against AD</description>
    <mitre>
      <id>T1110.003</id>
    </mitre>
    <group>authentication_failures,windows,attack,credential_access,</group>
  </rule>

</group>
```

**Key elements:**

| Element | Purpose |
|---------|---------|
| `<if_matched_sid>60122</if_matched_sid>` | Depends on the native Wazuh rule for Windows logon failures |
| `<same_field>win.eventdata.ipAddress</same_field>` | Correlates events sharing the same source IP — **required because `<same_source_ip />` does not work with the Windows EventChannel decoder** (see Lessons Learned) |
| `frequency=5` + `timeframe=1800` | 5 failures within 30 minutes triggers the alert |
| `level=13` | Critical severity — one level above SSH brute force (Rule 1, level 12) |

### Splunk SPL (Classification & Enrichment Query)

```spl
index=wazuh data.win.system.eventID=4625 earliest=-30m
| eval source_ip = 'data.win.eventdata.ipAddress'
| eval target_user = 'data.win.eventdata.targetUserName'
| stats 
    count as failed_attempts,
    dc(target_user) as unique_users_targeted,
    values(target_user) as users_list
  by source_ip, agent.name
| eval attempts_per_user = round(failed_attempts / unique_users_targeted, 2)
| eval attack_pattern = case(
    unique_users_targeted >= 5 AND attempts_per_user <= 3, "Password Spraying",
    unique_users_targeted = 1 AND failed_attempts >= 10, "Brute Force",
    unique_users_targeted >= 3 AND attempts_per_user > 3, "Mixed / Credential Stuffing",
    true(), "Unclassified"
  )
| where attack_pattern = "Password Spraying"
| sort - unique_users_targeted
| table source_ip, agent.name, failed_attempts, unique_users_targeted, attempts_per_user, attack_pattern, users_list
```

**Design notes:**
- The two `eval` statements at the top extract nested JSON fields to clean variable names using single-quote syntax. This is the working pattern for fields with dots in their names in Splunk.
- The classification logic in `case()` is the core of the detection: the ratio `attempts_per_user` mathematically distinguishes spraying from brute force regardless of total volume.
- `users_list` is preserved in the output — this is the blast radius information the SOC analyst needs to identify whether privileged accounts were targeted.

---

## Atomic Testing

### Test Command
Executed from Kali Linux (192.168.10.10) against the Domain Controller (192.168.10.50):

```bash
# Create user list file with domain users
cat > /tmp/users.txt << 'EOF'
jsmith
mgarcia
dbrown
swilson
mdavis
lmartinez
jjohnson
eanderson
rtaylor
lthomas
EOF

# Launch password spraying via SMB
crackmapexec smb 192.168.10.50 -u /tmp/users.txt -p 'WrongPassword2026!' --continue-on-success
```

### Expected Result

The Wazuh custom rule `100002` fires multiple times (one alert per correlation block of 5 events). The Splunk SPL query returns a row similar to:

| source_ip | agent.name | failed_attempts | unique_users | attempts/user | pattern |
|-----------|-----------|-----------------|--------------|---------------|---------|
| 192.168.10.10 | dc01-windows-server | 24 | 8 | 3.0 | Password Spraying |

The defining characteristic is `attempts_per_user ≈ 1.0-3.0` — each user was probed a small number of times with the same password, consistent with spraying behavior.

### Validation Commands

In Splunk:
```spl
index=wazuh rule.id=100002 earliest=-30m
| table _time, agent.name, rule.description, rule.level, rule.firedtimes
```

### Validation Screenshots

- Splunk SPL output showing automatic classification as "Password Spraying"

![Splunk SPL Output](/screenshots/phase6/02-password-spraying-splunk-classification.png)  

---

## False Positives

### Known FP Scenarios

| Scenario | Likelihood | Impact |
|----------|------------|--------|
| Sysadmin testing accounts after a password policy change | Medium | Generates FP from trusted IPs |
| Authorized account audit tools (PingCastle, Purple Knight, BloodHound legitimate use) | Low | Predictable — schedule whitelisted windows |
| Misconfigured service authenticating against multiple accounts with stale credentials | Medium | Could generate sustained FPs from internal IPs |
| Bulk account migration with mismatched credentials | Low-Medium | Whitelist during change window |
| Authorized penetration test | Low | Will trigger — coordinate suppression with red team |

### Mitigations

1. **Whitelist trusted internal IPs** with a follow-up suppression rule:
   ```xml
   <rule id="100003" level="0">
     <if_sid>100002</if_sid>
     <field name="win.eventdata.ipAddress">192.168.10.X</field>
     <description>Password spraying alert from whitelisted source - suppressed</description>
   </rule>
   ```

2. **Correlate with subsequent successful logons** — a password spraying alert followed by Event 4624 from the same source IP within minutes is a high-confidence compromise indicator. Could be implemented as a Rule 100003 in future iterations.

3. **Tune by department** — production servers and Domain Controllers should keep aggressive thresholds; jump hosts and test environments may need higher thresholds.

---

## Detection-Prevention Tradeoff

| Aspect | Status |
|--------|--------|
| **Detection** | This rule (100002) + Splunk SPL classification |
| **Prevention** | Mandatory MFA for all accounts; smart lockout policies; conditional access |
| **Why prevention is not always feasible** | Legacy accounts without MFA support, service accounts, on-premises systems without modern auth |
| **Recommended layered approach** | MFA + smart lockout + password policies blocking common passwords (e.g., HaveIBeenPwned API) + this detection rule + alert on successful logon following spraying pattern |

The most effective defense against password spraying is MFA — it neutralizes the attack entirely even when the password is correctly guessed. Detection serves as the safety net for accounts that cannot enforce MFA.

---

## Lessons Learned

This rule's development surfaced two important technical details that are not clearly documented elsewhere. Both are valuable for future Detection Engineering work.

### 1. Wazuh `<same_source_ip />` does not work with the Windows EventChannel decoder

The Wazuh element `<same_source_ip />` is shorthand for correlating events that share the `srcip` field. This field is populated by some decoders (notably the SSH/syslog decoder, where Rule 1 uses it successfully) but **not by the Windows EventChannel decoder**. For Windows events, the source IP is mapped to `data.win.eventdata.ipAddress`, which `<same_source_ip />` cannot reach.

**Correct pattern for Windows event correlation:**
```xml
<same_field>win.eventdata.ipAddress</same_field>
```

`<same_field>` is the generic form that explicitly references the field path to use for correlation. Always validate which decoder populates which fields before using shorthand correlation elements.

---

## References

- [MITRE ATT&CK T1110.003 — Password Spraying](https://attack.mitre.org/techniques/T1110/003/)
- [CISA AA24-057A — Russian SVR Cyber Actors Adapt Tactics for Initial Cloud Access](https://www.cisa.gov/news-events/cybersecurity-advisories/aa24-057a)
- [Microsoft — Midnight Blizzard: Guidance for responders on nation-state attack](https://www.microsoft.com/en-us/security/blog/2024/01/25/midnight-blizzard-guidance-for-responders-on-nation-state-attack/)
- [Wazuh Documentation — Custom Rules](https://documentation.wazuh.com/current/user-manual/ruleset/custom.html)
- [Atomic Red Team T1110.003](https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1110.003/T1110.003.md)
- [The Hacker Recipes — Password Spraying](https://www.thehacker.recipes/ad/movement/credentials/bruteforcing#password-spraying)

