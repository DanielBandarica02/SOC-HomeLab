# Rule [NUMBER]: [Rule Name]

## Metadata
| Field | Value |
|-------|-------|
| Rule ID | `custom-XXXXX` |
| Severity | Low / Medium / High / Critical |
| MITRE ATT&CK Tactic | [Tactic] |
| MITRE ATT&CK Technique | [TXXXX.XXX — Technique Name] |
| Data Source | Wazuh / Sysmon / auditd / pfSense |
| Platform | Linux / Windows / Network |
| Status | Active |

---

## Threat Context

### Description
[2-3 sentence summary of what this rule detects and why it matters.]

### Real-World Usage
[Which threat actors, malware families, or known campaigns use this technique.]

### Why This Matters
[Business or operational impact if this attack succeeds undetected.]

---

## Detection Strategy

### Logic
[Conceptual description of how the detection works. Plain English, no code.]

### Data Source Requirements
- Source: [e.g. `wazuh-alerts-*`]
- Required fields: [list of fields the rule depends on]
- Prerequisites: [e.g. Sysmon deployed, audit policy enabled]

### Thresholds
[Quantitative parameters: time window, event count, etc. Justification for each value.]

---

## Implementation

### Wazuh Rule (XML)
```xml
<group name="custom,...">
  <rule id="100001" level="X">
    ...
  </rule>
</group>
```

---

## Atomic Testing

### Test Command
```bash
[exact command to reproduce the attack from Kali Linux or target machine]
```

### Expected Result
[What the alert should look like — fields, severity, count]

### Validation Screenshot
![Rule validation](../screenshots/phase5/rule-XXXXX-validation.png)

---

## False Positives

### Known FP Scenarios
- [Scenario 1 — legitimate behavior that may trigger the rule]

### Mitigations
- [How to whitelist legitimate sources]
- [Threshold adjustments]

---

## References
- [MITRE ATT&CK technique URL]
- [Threat intelligence report URL]
- [Other relevant documentation]
