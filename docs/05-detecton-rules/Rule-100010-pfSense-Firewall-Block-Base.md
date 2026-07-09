# Rule 100010: pfSense Firewall Block Base
 
## Metadata
| Field | Value |
|-------|-------|
| Rule ID | `100010` |
| Severity | Low |
| MITRE ATT&CK Tactic | N/A (base rule for specialisations) |
| MITRE ATT&CK Technique | N/A |
| Data Source | pfSense syslog via custom decoder |
| Platform | Network |
| Status | Active |
 
---
 
## Threat Context
 
### Description
Base rule that fires for every pfSense-decoded packet block event. It captures the raw signal of the firewall enforcing a deny policy and serves as the parent rule for all specialisation and aggregation rules that identify specific block patterns of security interest.
 
### Real-World Usage
Firewall block events are the foundational network telemetry of any perimeter defence. They are consumed by SIEMs both for operational alerting when the pattern is suspicious and for forensic reconstruction of historical activity. Attackers do not intentionally trigger this rule; it fires as a side effect of any denied traffic they generate, whether from reconnaissance, exploitation attempts, or misconfigured infrastructure.
 
### Why This Matters
Without a base rule capturing every block, specialisation rules cannot reference them via `<if_sid>` inheritance. The rule is deliberately low-severity because a single block event carries minimal operational meaning — a properly segmented network produces many blocks per hour of normal traffic. The operational value emerges from the patterns extracted by the derived rules that use this rule as their parent.
 
---
 
## Detection Strategy
 
### Logic
The rule fires when the custom pfSense decoder (`pfsense-custom-header`, defined in Phase 5 Part 4) successfully parses a syslog event and the extracted action field equals `block`. Level 3 severity places the rule below the dashboard's default operational threshold, ensuring individual blocks are retrievable via search but do not clutter alerting views.
 
### Data Source Requirements
- Source: pfSense syslog forwarded to Wazuh manager on UDP 514
- Required fields: `action` (extracted by custom decoder), `srcip`, `dstip`, `dstport`
- Prerequisites: pfSense syslog integration configured, custom decoder `pfsense-custom-header` deployed and validated
### Thresholds
Not applicable — the rule fires once per matched event without aggregation.
 
---
 
## Implementation
 
### Wazuh Rule (XML)
```xml
<group name="pfsense,custom,">
  <rule id="100010" level="3">
    <decoded_as>pfsense-custom-header</decoded_as>
    <match>block</match>
    <description>pfSense: firewall blocked a packet</description>
    <group>firewall,firewall_drop,</group>
  </rule>
</group>
```
 
---
 
## Atomic Testing
 
### Test Command
From Kali (VLAN 66) targeting a denied port on VLAN 20:
```bash
nmap -Pn -sS -p 10.10.20.10 445
```
 
### Expected Result
A single alert in `wazuh-alerts-*` with:
- `rule.id: 100010`
- `rule.level: 3`
- `rule.description: "pfSense: firewall blocked a packet"`
- `data.action: block`
- `data.srcip: 10.10.66.10`
- `data.dstip: 10.10.20.10`
- `data.dstport: 445`
### Validation Screenshot
![Rule 100010 validation](../screenshots/phase5/rule-100010-validation.png)
 
---
 
## False Positives
 
### Known FP Scenarios
- Legitimate traffic hitting default-deny rules during normal network operation — this is expected baseline noise in a segmented network, not a false positive in the traditional sense.
- Broadcast and multicast traffic reaching interfaces where it is not permitted — many protocols emit periodic frames that pfSense correctly blocks.
### Mitigations
- No mitigation required at this rule level; the low severity (3) prevents dashboard pollution.
- Specialisation rules (100011, 100012, 100013) apply context-specific filters to identify the subset of blocks that carry operational meaning.
---
 
## References
- [Wazuh documentation — Custom rules](https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html)
- [pfSense documentation — Firewall logging](https://docs.netgate.com/pfsense/en/latest/monitoring/logs/firewall.html)
- Internal reference: `docs/02-soc-stack/04-pfsense-syslog.md`
