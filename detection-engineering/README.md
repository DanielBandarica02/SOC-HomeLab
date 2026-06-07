# Detection Engineering

This directory contains the professional documentation for all custom detection rules deployed in the SOC HomeLab. Each rule is documented following a standardized Detection Engineering methodology covering threat modeling, detection strategy, implementation, atomic testing, and false positive analysis.

For ready-to-deploy rule code, see [`../rules/`](../rules/).
For the methodology overview, see [`../docs/phase6-detection-rules.md`](../docs/phase6-detection-rules.md).

---

## Rule Index

| # | Rule | MITRE | Tactic | Severity |
|---|------|-------|--------|----------|
| 1 | [SSH Brute Force](rules/01-ssh-bruteforce.md) | T1110.001 | Credential Access | High |
| 2 | [Password Spraying](rules/02-password-spraying.md) | T1110.003 | Credential Access | High |
| 3 | [RDP Brute Force](rules/03-rdp-bruteforce.md) | T1110.001 | Credential Access | High |
| 4 | [PowerShell Encoded Execution](rules/04-powershell-encoded.md) | T1059.001 | Execution | High |
| 5 | [Office Spawning Shell](rules/05-office-spawning-shell.md) | T1059 | Execution | Critical |
| 6 | [Scheduled Task Creation](rules/06-scheduled-task.md) | T1053.005 | Persistence | Medium |
| 7 | [Registry Autorun](rules/07-registry-autorun.md) | T1547.001 | Persistence | High |
| 8 | [Domain Admins Modification](rules/08-domain-admins.md) | T1098 | Privilege Escalation | Critical |
| 9 | [New Local Administrator](rules/09-new-local-admin.md) | T1136.001 | Persistence | High |
| 10 | [LSASS Memory Access](rules/10-lsass-access.md) | T1003.001 | Credential Access | Critical |
| 11 | [Kerberoasting](rules/11-kerberoasting.md) | T1558.003 | Credential Access | High |
| 12 | [Lateral Movement via SSH](rules/12-lateral-ssh.md) | T1021.004 | Lateral Movement | High |
| 13 | [Internal Nmap Scan](rules/13-internal-nmap.md) | T1046 | Discovery | Medium |
| 14 | [AD Enumeration](rules/14-ad-enumeration.md) | T1087.002 | Discovery | Medium |
| 15 | [Windows Defender Disabled](rules/15-defender-disabled.md) | T1562.001 | Defense Evasion | Critical |

---

## Methodology

Each rule follows the Detection Engineering lifecycle:

1. **Threat Modeling** — identifying adversary behavior and impact
2. **Detection Strategy** — defining the logic and data requirements
3. **Implementation** — coding in both Wazuh (XML) and Splunk (SPL)
4. **Atomic Testing** — validation through controlled attacks from Kali
5. **Documentation** — standardized template covering all aspects
6. **Tuning** — false positive analysis and refinement

See the [rule template](rule-template.md) for the documentation format used across all rules.
