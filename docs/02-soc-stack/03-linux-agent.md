# Phase 5 — SOC Stack (Part 3: Linux Agent + auditd)
 
## Overview
 
A fourth Wazuh agent was deployed on `ws-dev-02` (Ubuntu Desktop 24.04, VLAN 20, workgroup) to bring Linux telemetry into the SIEM platform alongside the three Windows agents from Part 2. The Linux endpoint completes the cross-OS coverage: one Domain Controller, two Windows workstations (corporate + dev), and one Linux developer station — four agents spanning two trust zones, two operating systems, and two identity models, all reporting to the same `wazuh-srv` manager.
 
The architectural contrast with the Windows agents is the central pedagogical theme of this document. **Windows requires an external component (Sysmon) to achieve EDR-grade visibility**: process creation, network connections per process, registry persistence, and file events all live in a separate channel that Sysmon adds. **Linux already exposes the same visibility natively through the kernel audit subsystem (`auditd`)** — no external instrumentation is needed, only configuration. The same SIEM agent (Wazuh) consumes both, demonstrating that the SIEM layer is independent of how the OS makes its events available.
 
This document covers the agent installation, the deployment of `auditd` with custom rules tuned for SOC L1 scenarios, and the **multi-source ingestion strategy** that emerged during deployment: instead of relying on a single log file with a single decoder, the agent monitors three complementary sources (`auth.log`, `syslog`, `audit.log`) each parsed by its appropriate Wazuh decoder. Six distinct gotchas surfaced during the deployment, all documented with methodology and solution; one limitation was accepted as a known constraint of the current Wazuh + Ubuntu 24.04 pairing.
 
---
 
## Architecture
 
```mermaid
flowchart LR
    subgraph wsdev2[ws-dev-02 — Ubuntu Desktop 24.04]
        kernel[Kernel audit subsystem]
        auditd[auditd daemon<br/>RAW format]
        audisp[audisp-syslog plugin]
        authlog[/var/log/auth.log]
        syslog_f[/var/log/syslog]
        auditlog[/var/log/audit/audit.log]
        pam[PAM / sudo]
        systemd[systemd services]
        agent[Wazuh agent<br/>3 localfile blocks]
        kernel --> auditd
        auditd --> auditlog
        auditd --> audisp
        audisp --> authlog
        pam --> authlog
        systemd --> syslog_f
        authlog --> agent
        syslog_f --> agent
        auditlog --> agent
    end
    pfsense[pfSense<br/>VLAN 20 → 99: TCP 1514]
    wazuh[wazuh-srv<br/>Manager + Indexer + Dashboard]
    agent -.->|TCP 1514| pfsense
    pfsense -.->|TCP 1514| wazuh
```
 
The three log sources are complementary, not redundant — each contributes a different perspective on the same events:
 
| Source                              | Perspective                  | Captures                                                              | Decoder        |
| ----------------------------------- | ---------------------------- | --------------------------------------------------------------------- | -------------- |
| `/var/log/auth.log`                 | PAM authentication layer     | sudo authentications, login attempts, SSH session lifecycle           | `pam`, `sudo`  |
| `/var/log/syslog`                   | systemd services             | service start/stop, cron, generic system messages                     | `syslog`       |
| `/var/log/audit/audit.log`          | kernel audit subsystem       | syscalls (execve, connect), file watches, config changes              | `auditd`       |
 
Maintaining all three gives defense-in-depth at the data plane level: if any single decoder has issues with a specific event type, the same security-relevant activity is likely captured by another source. This redundancy proved essential during deployment (see Troubleshooting #6).
 
---
 
## Deployment
 
### Pre-flight check — NTP and firewall
 
The same NTP discipline applied to `wazuh-srv` in Part 1 was applied to `ws-dev-02` before any Wazuh component was installed. Wazuh's internal PKI between agent and manager validates certificates against the host clock; a few minutes of drift causes silent enrollment failures.
 
```bash
timedatectl status
```
 
Expected: `NTP service: active` and `System clock synchronized: yes`. The clock was already synchronized via `systemd-timesyncd` (default in Ubuntu Desktop 24.04).
 
Connectivity to the manager was confirmed before triggering the wizard:
 
```bash
ping -c 3 10.10.99.10                    # replies, ICMP via pfSense
```
 
The pfSense rule `Pass VLAN20 net → 10.10.99.10 TCP 1514:1515` provisioned in Part 1 was verified to be in place before agent enrollment.
 
### Agent deployment via the Wazuh Dashboard wizard
 
The same wizard-driven approach used for the three Windows agents in Part 2 was used here, with **Linux DEB amd64** selected as the operating system. Wizard inputs:
 
| Field           | Value          |
| --------------- | -------------- |
| Operating system | Linux DEB amd64 |
| Server address  | `10.10.99.10`  |
| Agent name      | `ws-dev-02`    |
| Group           | `default`      |
 
The wizard generated three PowerShell-equivalent shell commands. Executed in order on `ws-dev-02` via SSH:
 
```bash
# 1. Download the agent .deb
curl -so wazuh-agent.deb \
  https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.x-1_amd64.deb
 
# 2. Install with manager and group preconfigured
sudo WAZUH_MANAGER='10.10.99.10' WAZUH_AGENT_NAME='ws-dev-02' \
     WAZUH_AGENT_GROUP='default' dpkg -i ./wazuh-agent.deb
 
# 3. Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
 
The agent reached `Active` state in `Server Management → Agents` within seconds:
 
| Name        | Status | IP             | Operating System | Version |
| ----------- | ------ | -------------- | ---------------- | ------- |
| ws-dev-02   | Active | `10.10.20.20`  | Ubuntu 24.04     | 4.14.x  |
 
This is the moment where the cross-VLAN enrollment is validated for Linux as well — the path provisioned in Part 1 (`Pass VLAN20 → 10.10.99.10:1514-1515`) now carries telemetry from a fourth host living in a separate trust zone.
 
### Initial validation — baseline coverage from default `<localfile>` blocks
 
Before installing auditd, baseline validation was performed using the localfile blocks the Wazuh agent ships with by default (`/var/log/dpkg.log`, `/var/ossec/logs/active-responses.log`, and the `journald` block). A synthetic failed-sudo event was generated:
 
```bash
sudo -k                    # invalidate sudo cache
sudo --user=nobody true    # prompts for password; three wrong attempts
```
 
The corresponding PAM authentication failure events appeared in the dashboard within 10 seconds, confirming the agent is forwarding telemetry end-to-end before any custom configuration. This baseline check is what surfaces a broken pipeline early — before the operator starts adding rules and decoders that complicate diagnosis.
 
### auditd installation and configuration
 
`auditd` and its plugin suite were installed:
 
```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
sudo systemctl status auditd
```
 
The default Ubuntu 24.04 configuration writes audit events in `ENRICHED` format, which (as documented in Troubleshooting #3) breaks Wazuh's audit decoder. The format was changed to `RAW`:
 
```bash
sudo sed -i 's/^log_format = ENRICHED/log_format = RAW/' /etc/audit/auditd.conf
sudo grep "^log_format" /etc/audit/auditd.conf
```
 
The default `log_group = adm` was kept rather than switching to `log_group = wazuh`. To allow the Wazuh agent (which runs as user `wazuh`) to read `/var/log/audit/audit.log`, the `wazuh` user was added to the `adm` group:
 
```bash
sudo usermod -aG adm wazuh
groups wazuh                # should include adm
```
 
This decision keeps auditd in its standard Ubuntu configuration and grants read access to wazuh by group membership — see Troubleshooting #1 for the diagnostic path that led here.
 
The audisp-syslog plugin was activated so that auditd events are also forwarded to syslog, providing a second ingestion path through `/var/log/auth.log`:
 
```bash
sudo sed -i 's/^active = no/active = yes/' /etc/audit/plugins.d/syslog.conf
sudo cat /etc/audit/plugins.d/syslog.conf | grep "^active"
```
 
auditd was restarted to apply all changes:
 
```bash
sudo systemctl stop auditd
sleep 2
sudo systemctl start auditd
sudo systemctl status auditd | head -5
```
 
### Custom audit rules — SOC L1 ruleset
 
A curated rule set was placed in `/etc/audit/rules.d/soc-monitoring.rules`. The intent was a small, focused set covering privilege escalation and identity file modifications — broad enough to demonstrate detection capabilities, narrow enough to avoid the noise that comes with shipping STIG-style rulesets.
 
```
## Privilege escalation: any command run as root from a non-root user
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=-1 -k priv_esc
 
## Identity file modifications
-w /etc/passwd -p wa -k identity_files
-w /etc/shadow -p wa -k identity_files
-w /etc/sudoers -p wa -k identity_files
```
 
Rules were loaded into the kernel:
 
```bash
sudo augenrules --load
sudo auditctl -l
```
 
`auditctl -l` confirmed the rules are active. The same key strings (`priv_esc`, `identity_files`) appear in audit events when the corresponding activity occurs, making the events filterable in the SIEM by content.
 
### Wazuh agent configuration — three complementary `<localfile>` blocks
 
This is where the deployment took its current shape. After the debug paths documented in Troubleshooting #2 through #6, the working configuration uses **three log sources in parallel**, each with the correct decoder:
 
`/var/ossec/etc/ossec.conf` was extended with three blocks (added inside `<ossec_config>`):
 
```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
  <only-future-events>no</only-future-events>
</localfile>
 
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
 
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/syslog</location>
</localfile>
```
 
The `only-future-events: no` setting on the audit.log block forces the agent to read from the beginning of the file on restart, ensuring no events are missed when the agent service cycles.
 
After validating the XML and restarting the agent:
 
```bash
sudo xmllint --noout /var/ossec/etc/ossec.conf && echo "XML OK"
sudo systemctl stop wazuh-agent
sleep 3
sudo find /var/ossec/queue -name "*.state" -delete 2>/dev/null
sudo systemctl start wazuh-agent
sleep 10
sudo grep "Analyzing file" /var/ossec/logs/ossec.log | tail -5
```
 
The agent log confirmed it was monitoring all three new sources:
 
```
INFO: (1950): Analyzing file: '/var/log/audit/audit.log'.
INFO: (1950): Analyzing file: '/var/log/auth.log'.
INFO: (1950): Analyzing file: '/var/log/syslog'.
```
 
---
 
## Validation
 
### Agent health in the dashboard
 
`Server Management → Agents` confirmed `ws-dev-02` joined the existing roster:
 
| Name        | Status | IP             | OS                            |
| ----------- | ------ | -------------- | ----------------------------- |
| DC01        | Active | `10.10.10.10`  | Microsoft Windows Server 2022 |
| WS-CORP-01  | Active | `10.10.10.20`  | Microsoft Windows 11 Pro      |
| WS-DEV-01   | Active | `10.10.20.10`  | Microsoft Windows 11 Pro      |
| **ws-dev-02** | **Active** | **`10.10.20.20`** | **Ubuntu 24.04**          |
 
Four agents, two operating systems, two VLANs, two identity models — the SIEM data plane is operational across the lab.
 
### Multi-source telemetry — three decoders working in parallel
 
Three dashboard queries confirmed each ingestion path is delivering events:
 
```
agent.name: "ws-dev-02" and decoder.name: "auditd"        # 258 hits
agent.name: "ws-dev-02" and decoder.name: "sudo"          # 136 hits
agent.name: "ws-dev-02" and full_log: "priv_esc"          # 30 hits
```
 
Each result represents a distinct view of the same host:
- The `auditd` decoder processes events from `/var/log/audit/audit.log` (CONFIG_CHANGE, AVC, DAEMON_*).
- The `sudo` decoder processes events from `/var/log/auth.log` (PAM session lifecycle, sudo authentications).
- Content searches on `full_log` retrieve SYSCALL events containing the rule keys, even when the auditd decoder doesn't fully extract them (see Troubleshooting #6).
### Synthetic event — privilege escalation
 
```bash
sudo -k
sudo whoami
```
 
This dispatches several events across all three pipelines simultaneously:
- A `type=SYSCALL ... key="priv_esc"` event in `/var/log/audit/audit.log`
- A `pam_unix(sudo:session): session opened` event in `/var/log/auth.log`
- A `sudo: arodriguez : ... COMMAND=/usr/bin/whoami` event in `/var/log/auth.log`
Within 15 seconds, the dashboard showed:
- `decoder.name: sudo` events for the sudo session (PAM + command logging)
- `full_log: priv_esc` events for the auditd SYSCALL captures
- Rule mapping to MITRE T1078 (Valid Accounts) for the PAM session events
### Synthetic event — identity file modification
 
```bash
sudo bash -c 'echo "# test comment $(date)" >> /etc/passwd'
sudo head -3 /etc/passwd
sudo sed -i '/# test comment/d' /etc/passwd
```
 
The first command writes to `/etc/passwd`, the third undoes it. Both writes trigger the `identity_files` audit rule. The dashboard filter `agent.name: "ws-dev-02" and full_log: "identity_files"` returned events with timestamps matching the actions, demonstrating that the file-watch rules dispatch correctly through the pipeline.
 
### Agent ossec.log audit
 
`/var/ossec/logs/ossec.log` was inspected after the validation run to confirm no buffer drops or queue saturation:
 
```bash
sudo grep -iE "queue|drop|buffer" /var/ossec/logs/ossec.log | tail -10
```
 
The only line returned was the standard buffer status:
 
```
wazuh-agentd: INFO: Buffer agent.conf updated, enable: 1 size: 5000
```
 
No drop events occurred — the volume of telemetry produced during validation stayed comfortably within the agent's 5000-event buffer.
 
---
 
## Troubleshooting & Lessons Learned
 
### 1. auditd writes `/var/log/audit/audit.log` with restrictive permissions by default
 
After installing auditd, the agent could see the file via the `<localfile>` block but no events were forwarded to the SIEM. The agent log showed `Analyzing file: '/var/log/audit/audit.log'`, but no events with `decoder.name: auditd` reached the dashboard.
 
The methodology:
 
| Test                                                  | Result                                | Layer eliminated                |
| ----------------------------------------------------- | ------------------------------------- | ------------------------------- |
| `ls -la /var/log/audit/audit.log`                     | `-rw-------  root root`                | File mode `600`, only root can read |
| `sudo -u wazuh head -5 /var/log/audit/audit.log`      | `Permission denied`                   | Agent (running as `wazuh`) cannot read |
 
**Root cause:** auditd writes its log with mode `600` and group `root` by default. The Wazuh agent runs as user `wazuh:wazuh`, which has no read access to a `600 root root` file. The agent reads the directory entry but every read attempt on the content fails silently — the agent does not surface this as a prominent error.
 
**Solution:** add the `wazuh` user to the `adm` group (Ubuntu's traditional log-readers group) and change auditd's `log_group` to `adm`:
 
```bash
sudo usermod -aG adm wazuh
sudo sed -i 's/^log_group = root/log_group = adm/' /etc/audit/auditd.conf
sudo systemctl stop auditd && sudo systemctl start auditd
sudo systemctl restart wazuh-agent
```
 
After the change, `ls -la /var/log/audit/audit.log` showed `-rw-r----- root adm`, and `sudo -u wazuh head` returned content. Events began reaching the SIEM with `decoder.name: auditd`.
 
**Lesson:** when a log shipper appears to be working (file is being read, no errors in the agent log) but no events surface in the SIEM, file permissions are a common silent failure mode. Always verify with `sudo -u <shipper-user>` that the user can actually read the file content, not just see it in the directory listing.
 
### 2. journald takes precedence over the auditd `<localfile>` block
 
After fixing permissions and initial Wazuh agent configuration, sudo events were arriving at the SIEM but **decoded as `pam` from journald**, not as `auditd` events with `data.audit.*` populated. The auditd block in `ossec.conf` was loaded by the agent (`Analyzing file: '/var/log/audit/audit.log'` visible in `ossec.log`) but its events were apparently being captured by the journald block first.
 
The methodology:
 
| Test                                                  | Result                                | Layer eliminated                |
| ----------------------------------------------------- | ------------------------------------- | ------------------------------- |
| Inspect a dashboard event for sudo activity           | `decoder.name: pam`, `location: journald` | Decoder is journald-routed |
| Inspect `ossec.conf` for ingestion ordering           | journald block listed before audit.log block | Pipeline conflict identified |
 
**Root cause:** Ubuntu 24.04 routes auditd events into journald in addition to the standard audit.log file. Wazuh agent's journald reader processes events first and the agent considers them handled, so the `audit.log` block becomes redundant — its events do not surface as auditd-decoded entries because journald already classified them as PAM events.
 
**Solution:** remove the journald `<localfile>` block from `ossec.conf` rather than filtering it (filters on journald are fragile in Wazuh). After restart, sudo events appeared via `auth.log` (with the `sudo` decoder) and via `audit.log` (with the `auditd` decoder) — the two correct routes.
 
**Trade-off accepted:** journald is not monitored anymore. systemd service status events (wazuh-agent restarts, cron job starts, etc.) no longer reach the SIEM through this single path. This is acceptable because `/var/log/syslog` (added as a separate `<localfile>` block) covers the same ground for systemd-managed services.
 
**Lesson:** when two ingestion sources cover overlapping events with different decoders, the SIEM applies whichever it processes first. Removing the redundancy is cleaner than filtering inside the conflicting source. **Simplification beats configuration acrobatics**.
 
### 3. auditd `log_format = ENRICHED` breaks Wazuh's decoder
 
Even after fixing permissions and removing the journald block, SYSCALL events from auditd still didn't surface with `data.audit.*` populated. The agent was reading the file, the decoder was loaded, but the structured fields stayed empty.
 
The methodology:
 
| Test                                                  | Result                                | Layer eliminated                |
| ----------------------------------------------------- | ------------------------------------- | ------------------------------- |
| `sudo head -3 /var/log/audit/audit.log`               | Fields like `res=successAUID="..."` concatenated without separators | Output format anomaly identified |
| `grep "^log_format" /etc/audit/auditd.conf`           | `log_format = ENRICHED`               | Configuration source confirmed  |
 
**Root cause:** Ubuntu 24.04 ships auditd with `log_format = ENRICHED` as default. The enriched format appends user/role context to each event (`AUID="..." UID="root" GID="root"`) at the end of the line, **without a clean separator** from the preceding field:
 
```
... success=yes ... res=successAUID="unset" UID="root"
```
 
Wazuh's regex-based audit decoder assumes the classic `RAW` format with clean field termination. The enriched output is partially matched and the structured extraction fails — the event reaches the SIEM but with empty `data.audit.*` fields.
 
**Solution:** change auditd to RAW format in `/etc/audit/auditd.conf`:
 
```bash
sudo sed -i 's/^log_format = ENRICHED/log_format = RAW/' /etc/audit/auditd.conf
sudo systemctl stop auditd && sudo systemctl start auditd
```
 
After the change, new events appeared with the classic RAW format — no concatenated AUID/UID at the end. Note that **existing entries in audit.log retain the old enriched format** until log rotation; only events written after the auditd restart are in RAW.
 
**Lesson:** when a log shipper "reads" a file but produces no parsed events, the bug is rarely in the shipper — it's in the format mismatch between source and decoder. A single `head` of the source log reveals what would otherwise require enabling debug-level decoder traces.
 
### 4. XML typo in `<ossec_config>` blocks the agent from starting
 
After several rounds of editing `ossec.conf` to remove and re-add `<localfile>` blocks, a restart of the agent failed with `Job for wazuh-agent.service failed because the control process exited with error code` and a reference to "line 0 of /var/ossec/etc/ossec.conf".
 
The methodology:
 
| Test                                                  | Result                                | Layer eliminated                |
| ----------------------------------------------------- | ------------------------------------- | ------------------------------- |
| `sudo xmllint --noout /var/ossec/etc/ossec.conf`      | Parse error referencing the opening tag | XML malformed                  |
| `grep -n "ossec_confing\|ossec_config"`               | Line 190: `<ossec_confing>` (typo)    | Root cause identified           |
 
**Root cause:** during one of the edit cycles, the opening tag was typed as `<ossec_confing>` (missing the second `g` from "config" and adding an extra `n`). The XML became unparseable because the closing tag `</ossec_config>` had no matching opening tag.
 
**Solution:** correct the typo to `<ossec_config>`. Validated with `xmllint --noout`, which returned no errors.
 
**Lesson:** when an editor reports "line 0" or a similar non-specific location, the XML parser has rejected the entire document and cannot point to a meaningful position. `xmllint --noout` against the config file is the fastest way to diagnose — it reports the exact line and character of the syntax error. Run it as a habit after every manual edit of large XML configurations.
 
### 5. Three `<localfile>` blocks complementary, not one
 
After resolving the previous gotchas, the pipeline still had gaps — some events (sudo authentications, SSH sessions) arrived via journald (which was now removed), while auditd's `audit.log` events struggled with SYSCALL decoding.
 
**Root cause:** the Wazuh agent's default installation includes `<localfile>` blocks for `dpkg.log`, `active-responses.log`, and `journald`, but **not for `/var/log/auth.log`, `/var/log/syslog`, or `/var/log/audit/audit.log`**. These are deliberately left for the operator to configure based on the deployment's needs.
 
The temptation when integrating auditd is to add only the audit.log block and assume the audit decoder covers the entire Linux security telemetry surface. This is a misconception: PAM/sudo events live in `auth.log` and are best decoded by Wazuh's PAM/sudo decoders; systemd service events live in `syslog` and are best decoded by the generic syslog decoders. The audit.log captures kernel syscalls with the auditd decoder. **The three are complementary, not redundant.**
 
**Solution:** add three `<localfile>` blocks, one per source, each with its appropriate decoder format:
 
```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
  <only-future-events>no</only-future-events>
</localfile>
 
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
 
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/syslog</location>
</localfile>
```
 
After adding the three blocks and restarting the agent, the dashboard immediately showed events from all three sources with their respective decoders applied (`auditd`, `sudo`, `syslog`).
 
**Lesson:** Linux security telemetry is distributed across multiple log files by design — PAM, systemd, and the kernel audit subsystem each write to their own location. A Wazuh agent configuration that only reads one of them captures a narrow slice of the security picture. **Defense-in-depth applies to data ingestion as much as to network segmentation**: multiple complementary sources mitigate the risk that any single decoder or pipeline has issues with a specific event type.
 
### 6. `data.audit.key` not extracted from SYSCALL events — accepted limitation
 
After resolving all previous gotchas, a final observation: SYSCALL events from auditd reach the SIEM with the full event text in `full_log` (including `key="priv_esc"`), but the structured field `data.audit.key` remains empty for SYSCALL events. Other audit event types (`CONFIG_CHANGE`, `DAEMON_*`, `AVC`) populate `data.audit.*` correctly.
 
The methodology:
 
| Verification                                                     | Result                              |
| ----------------------------------------------------------------- | ----------------------------------- |
| `sudo grep "key=\"priv_esc\"" /var/log/audit/audit.log \| wc -l` | 280+ matches — rule firing locally   |
| Dashboard: `agent.name: "ws-dev-02" and full_log: "priv_esc"`     | 30 events — pipeline transports them |
| Dashboard: `agent.name: "ws-dev-02" and data.audit.key: "priv_esc"` | 0 events — structured field empty  |
 
**Diagnosis:** the auditd decoder's regex in Wazuh 4.14 paired with auditd RAW format on Ubuntu 24.04 does not extract the `key` field from multi-line SYSCALL events. The event is forwarded with all data in `full_log`, but the `data.audit.*` mapping for SYSCALL stays empty. This is a known limitation in this pairing.
 
**Operational impact:** searches by content (`full_log: "priv_esc"`) work; searches by structured field (`data.audit.key: "priv_esc"`) do not. Wazuh built-in rules that depend on the decoded `audit.key` field do not match these events. Custom detection rules using `full_log` regex matching are unaffected.
 
**Trade-off accepted (lab decision):** the limitation is acknowledged but not resolved with custom decoder modifications. Reasons:
 
1. **Functional coverage exists.** Events reach the SIEM with all relevant data in `full_log`, making them queryable and triageable.
2. **Complementary sources cover the gap.** `/var/log/auth.log` and `/var/log/syslog` capture sudo, PAM, and systemd events with **full structured decoding** via their respective Wazuh decoders.
3. **A future iteration** could integrate the `audisp-json` plugin (which emits auditd events as native JSON, consumable by Wazuh with `log_format: json` and fully structured). This is documented as a roadmap item.
**Lesson:** when a SIEM ingestion pipeline has multiple decoders covering overlapping ground, incomplete parsing in one decoder is mitigated by the others. This is exactly why the multi-source configuration in Troubleshooting #5 matters — it transforms what would be a blocking limitation into an accepted trade-off with documented mitigations.
 
---
 
## Result
 
- Wazuh agent deployed on `ws-dev-02` (Ubuntu Desktop 24.04, `10.10.20.20/24`, VLAN 20, workgroup standalone), `Active` in the dashboard alongside the three Windows agents from Part 2.
- Cross-VLAN enrollment path validated: VLAN 20 → VLAN 99 over TCP 1514/1515, traversing the pfSense allow rule provisioned in Part 1.
- auditd installed and configured: `log_format = RAW`, `log_group = adm`, audisp-syslog plugin active.
- `wazuh` user added to `adm` group to enable read access to `/var/log/audit/audit.log`.
- Custom audit ruleset in `/etc/audit/rules.d/soc-monitoring.rules` with two active keys (`priv_esc`, `identity_files`); kernel rules verified loaded via `auditctl -l`.
- Three complementary `<localfile>` blocks added to `ossec.conf`: `audit.log` (audit decoder), `auth.log` (syslog format → pam/sudo decoders), `syslog` (syslog decoder). Default `journald` block removed to avoid decoder conflicts.
- Telemetry validated from three sources in parallel: 258 events via `auditd` decoder, 136 events via `sudo` decoder, 30 events with `priv_esc` content in `full_log` — all from the same host.
- Synthetic events validated: `sudo whoami` produces priv_esc auditd events and PAM session events; modifications to `/etc/passwd` trigger identity_files audit events.
- Limitation documented: `data.audit.key` not consistently extracted from SYSCALL events in Wazuh 4.14 + Ubuntu 24.04. Content-based queries on `full_log` work; structured-field queries on `data.audit.key` do not. The complementary `auth.log`/`syslog` sources mitigate this gap with full structured parsing.
- Six distinct troubleshooting entries surfaced and documented with methodology: file permissions, journald precedence, auditd format mismatch, XML typo, multi-source necessity, and the SYSCALL decoding limitation.
- Snapshot: `ws-dev-02-baseline-soc` taken in VirtualBox after the full pipeline was validated, capturing the working state of agent + auditd + rules + ossec.conf.
---
 
## Screenshots
 
| Screenshot | Description |
| ---------- | ----------- |
| ![ws-dev-02 active in dashboard](../Screenshots/phase5/26-ws-dev-02-agent-active.png) | `Server Management → Agents` showing the fourth agent connected |
| ![auditd status running](../Screenshots/phase5/27-auditd-status.png) | `systemctl status auditd` after install, RAW format applied |
| ![auditctl -l with custom rules](../Screenshots/phase5/28-auditctl-l-rules.png) | `auditctl -l` showing the priv_esc and identity_files rules loaded |
| ![Wazuh user in adm group](../Screenshots/phase5/29-wazuh-in-adm.png) | `groups wazuh` confirming adm membership for audit.log read access |
| ![ossec.conf three localfile blocks](../Screenshots/phase5/30-ossec-three-localfile.png) | The three complementary `<localfile>` blocks added to the agent config |
| ![Analyzing file in ossec.log](../Screenshots/phase5/31-analyzing-three-sources.png) | Agent log confirming it monitors auth.log, syslog, and audit.log |
| ![Dashboard auditd decoder](../Screenshots/phase5/32-dashboard-auditd-decoder.png) | Filter `decoder.name: auditd` showing 258 events for ws-dev-02 |
| ![Dashboard sudo decoder](../Screenshots/phase5/33-dashboard-sudo-decoder.png) | Filter `decoder.name: sudo` showing 136 events from PAM |
| ![Dashboard priv_esc content](../Screenshots/phase5/34-dashboard-priv-esc-fulllog.png) | Filter `full_log: priv_esc` showing 30 SYSCALL events with the rule key |
| ![MITRE T1078 mapping](../Screenshots/phase5/35-mitre-t1078-pam.png) | A PAM session event mapped to MITRE Valid Accounts (T1078) |
| ![Four agents complete roster](../Screenshots/phase5/36-four-agents-roster.png) | The final agent roster: DC01, WS-CORP-01, WS-DEV-01, ws-dev-02, all Active |
 
---
 
*Previous: [Phase 5 — SOC Stack (Part 2: Windows Agents + Sysmon)](02-windows-agents.md)*
*Next: [Phase 5 — SOC Stack (Part 4: pfSense Syslog Integration)](04-pfsense-syslog.md)*
