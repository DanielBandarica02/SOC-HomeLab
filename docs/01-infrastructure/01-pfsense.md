# Phase 1.1 — Perimeter: pfSense + OpenVPN
 
> **Goal:** pfSense running as the single inter-VLAN router for the lab, with Internet egress, DHCP per LAN, OpenVPN as the only sanctioned crossing between VLAN 10 and VLAN 20, and a dedicated management path for me to administer the firewall before any other VM exists.

---
 
## 1. Create the VirtualBox VM
 
### General settings
 
| Field   | Value                |
| ------- | -------------------- |
| Name    | `pfSense`            |
| Type    | BSD                  |
| Version | FreeBSD (64-bit)     |
 
### Resources
 
| Resource | Value |
| -------- | ----- |
| CPU      | 2 vCPU |
| RAM      | 2 GB |
| Disk     | 20 GB |

 
## 2. Install pfSense

The pfSense VM was created with five virtual NICs (1 NAT + 4 Internal Network) to provide one interface per VLAN plus the WAN uplink. All adapters use **Intel PRO/1000 MT Desktop (82540EM)** because FreeBSD has native drivers for this NIC.

| Adapter | Attached to                         | Promiscuous Mode |
| ------- | ----------------------------------- | ---------------- |
| 1 (em0) | NAT                                 | Deny             |
| 2 (em1) | Internal Network `vlan10-corp`      | Allow VMs        |
| 3 (em2) | Internal Network `vlan20-dev`       | Allow VMs        |
| 4 (em3) | Internal Network `vlan66-attack`    | Allow VMs        |
| 5 (em4) | Internal Network `vlan99-soc`       | Allow VMs        |

![Network Adapters](screenshots/phase2/01-network-adapters.png)


For every adapter:
 
- **Promiscuous Mode:** `Allow All` — required for Suricata to inspect everything traversing the firewall, and for visibility into broadcast / multicast traffic (ARP poisoning, LLMNR / NBT-NS abuse, etc.) that's central to many MITRE techniques. Without it the IDS would be partially blind. 

### pfSense installation

pfSense 2.8.1 was installed from the Netgate Installer ISO 

### Interface assignment and IP configuration

From the pfSense console, interfaces were assigned (em0=WAN, em1=LAN, em2=OPT1, em3=OPT2, em4=OPT3) and IP addresses configured per the lab IP plan:

| Interface | IPv4 / mask     | DHCP server                |
| --------- | --------------- | -------------------------- |
| WAN       | DHCP — 10.0.2.15 | n/a                       |
| LAN       | 10.10.10.1/24    | Enabled (.100-.200)       |
| OPT1      | 10.10.20.1/24    | Enabled (.100-.200)       |
| OPT2      | 10.10.66.1/24    | Disabled (static Kali)    |
| OPT3      | 10.10.99.1/24    | Disabled (static SOC VMs) |
 
 
### Verify with `ping 8.8.8.8`
 
Console option `7) Ping host` → `8.8.8.8`. Four replies → outbound NAT works. pfSense CE defaults to **Automatic Outbound NAT**, which generates NAT rules for every LAN interface without configuration.

![Ping Verification](screenshots/phase1/02-ping-verification.png)
 
---
 
## 3. Management interface (host-only) for Web UI access

I add a 6th adapter on a VirtualBox host-only network so I can access the GUI from my host PC.

 
### 4.4 The default-deny gotcha and the first Web UI access
 
At this point pfSense tells me I can access the Web UI at `https://192.168.56.10`. **It doesn't load.** This is the second real gotcha.
 
pfSense applies **default deny on every interface except LAN**. LAN has an "anti-lockout rule" automatically — every other interface, including the freshly-created MGMT, blocks everything inbound. The Web UI is reachable network-wise but the firewall drops the connection.
 
To break in, console option `8) Shell`:
 
```sh
pfctl -d         # disables the packet filter entirely
```
 
Now the browser loads `https://192.168.56.10`. Login `admin` / `pfsense`. I immediately go to:
 
**Firewall → Rules → OPT4 → Add** and create this rule:
 
| Field | Value |
| --- | --- |
| Action | Pass |
| Interface | OPT4 |
| Address Family | IPv4 |
| Protocol | **any** |
| Source | Single host or alias → `192.168.56.1` |
| Destination | **This Firewall (self)** |
| Description | `Allow host PC full admin access to pfSense` |
 
Save → Apply Changes.
 
Then back to the console shell:
 
```sh
pfctl -e         # re-enable the packet filter
exit
```
 
**Why `any` protocol and not just TCP/443 for HTTPS:** I started with the tighter rule (TCP/443 only) and immediately found myself wanting ping for diagnostics and the option to use SSH later. The "any" rule from `192.168.56.1` to `self` covers all admin paths without opening the firewall to broader traffic — destination is still pfSense itself, not a route into the lab.
 
**Why `pfctl -d` isn't a long-term solution:** pfSense auto-re-enables the packet filter on `Apply Changes`, service restarts, and several other events. The permanent rule is the right fix; `pfctl -d` is just for the first-access window.
 
---
 
## 5. Setup Wizard
 
Visiting the Web UI for the first time triggers pfSense's Setup Wizard. I walk it through with these values:
 
### General Information
 
| Field | Value | Note |
| --- | --- | --- |
| Hostname | `pfSense` | — |
| Domain | `soclab.internal` | **Not** `.local` — that TLD is reserved by mDNS (Avahi, Bonjour, etc.) and pfSense itself warns against it. `.internal` is the IETF / ICANN convention for private networks. |
| Primary DNS Server | `1.1.1.1` | Cloudflare |
| Secondary DNS Server | `8.8.8.8` | Google — mixing providers for redundancy |
| **Override DNS** | **unchecked** | Confusingly worded checkbox. Unchecked = my manual DNS values stay, DHCP from WAN doesn't override them. Checked = DHCP can override (we don't want that). |
 
### Time Server Information
 
| Field | Value |
| --- | --- |
| Timezone | `Europe/Madrid` |
| NTP Server | `pool.ntp.org` (default) |
 
### WAN Configuration
 
Mostly defaults (DHCP). The two checkboxes I have to **uncheck** at the bottom:
 
- **Block RFC1918 Private Networks** → uncheck
- **Block bogon networks** → uncheck
**Why:** my WAN sits on VirtualBox NAT (`10.0.2.0/24`), which is itself RFC1918. If I leave these blocks on, pfSense drops traffic from `10.0.2.2` (the VirtualBox NAT gateway) — including periodic DHCP renewals — and I lose WAN intermittently. In a real-world deployment with a public WAN, these checkboxes are good practice; on a virtualized NAT WAN, they're self-sabotage.
 
### LAN Configuration
 
Already set during the install wizard. Just verify `10.10.10.1/24` and click Next.
 
### Admin password
 
Change from `pfsense` to something strong. Save somewhere I can find it.
 
Reload → Finish → land on Dashboard.
 
---
 
## 6. Rename interfaces
 
Default names (`LAN`, `OPT1`, `OPT2`, `OPT3`, `OPT4`) are useless once I have rules and logs. I rename them to match the topology vocabulary so a future rule like `VLAN66 → VLAN99 blocked` immediately tells me "the attacker tried to touch the SIEM."
 
| Default name | Renamed to | Description |
| --- | --- | --- |
| LAN | **VLAN10** | Corporate domain segment |
| OPT1 | **VLAN20** | Software Development |
| OPT2 | **VLAN66** | Attacker DMZ |
| OPT3 | **VLAN99** | SOC Management (out-of-band) |
| OPT4 | **MGMT** | Temporary host-only admin |
 
For each interface: **Interfaces → [name] → Description field → Save.**
 
**Crucial:** Save all 5 first, then click **Apply Changes** once at the very end. The yellow "you must apply" banner accumulates across navigations. If I Apply after each rename, that's 5 `pfctl` reloads — and each reload can drop my browser session. One Apply at the end = one reload, one risk window.
 
After Apply, the firewall rules tabs and log labels all update to the new names. The existing OPT4 rule moves to the MGMT tab automatically — pfSense references interfaces by internal ID (`opt4`), not by display name.
 
---
 
## 7. OpenVPN — full setup
 
The architectural decision behind this: VLAN 10 (Corporate) and VLAN 20 (Software Development) are separate trust zones, and the only sanctioned crossing is an authenticated, logged OpenVPN tunnel. Direct firewall rules between the two segments stay denied. See [design-decisions.md §9](../00-architecture/design-decisions.md#9-why-openvpn-as-the-bridge-between-vlan-10-and-vlan-20) for the rationale.
 
### 7.1 Install the OpenVPN Client Export package
 
**System → Package Manager → Available Packages**.
 
Search for `openvpn-client-export`. The package "OpenVPN Client Export Utility" appears. **+ Install → Confirm**. Wait 30–60 seconds.
 
After install, **VPN → OpenVPN** has a new tab: `Client Export`.
 
### 7.2 Create the Certificate Authority
 
**System → Cert Manager → CAs → Add.**
 
This CA is the trust root of my lab VPN. It signs the server certificate and every user certificate. In production, this would be either a public CA or an enterprise PKI; in the lab, I am my own CA.
 
| Field | Value |
| --- | --- |
| Descriptive name | `SOC-Lab-CA` |
| Method | **Create an internal Certificate Authority** |
| Randomize Serial | ✓ |
| Key Type | RSA |
| Key Length | 2048 |
| Digest Algorithm | SHA256 |
| Lifetime (days) | 3650 |
| Common Name | `SOC-Lab-CA` |
| Country Code | `ES` |
| State or Province | `Catalonia` |
| City | `Girona` |
| Organization | `SOC HomeLab` |
| OU / Email | (blank) |
 
Save.
 
**Why 2048 bits and SHA256:** 2048 is the security/performance balance — 4096 doubles the key handshake cost for negligible practical security gain on a lab. SHA1 is broken, MD5 is more broken; SHA256 is the minimum acceptable today and the OpenVPN community consensus.
 
### 7.3 Create the server certificate
 
**System → Cert Manager → Certificates → Add/Sign.**
 
| Field | Value |
| --- | --- |
| Method | Create an internal Certificate |
| Descriptive name | `SOC-Lab-VPN-Server` |
| Certificate authority | **SOC-Lab-CA** |
| Key Type | RSA |
| Key Length | 2048 |
| Digest Algorithm | SHA256 |
| Lifetime (days) | 3650 |
| Common Name | `vpn.soclab.internal` |
| Country / State / City / Org | same as the CA |
| **Certificate Type** | **Server Certificate** |
| Alternative Names | (blank) |
 
Save.
 
**Why "Server Certificate":** the `extendedKeyUsage` X.509 extension differs between user certs and server certs. If I leave this as "User Certificate" (the default), the server will function but clients will emit warnings about an unexpected certificate type on every connect.
 
### 7.4 Create the OpenVPN server
 
**VPN → OpenVPN → Servers → Add.**
 
#### General Information
 
| Field | Value |
| --- | --- |
| Server Mode | **Remote Access (SSL/TLS + User Auth)** |
| Backend for authentication | Local Database |
| Device Mode | tun (Layer 3 Tunnel Mode) |
| Protocol | **UDP on IPv4 only** |
| Interface | **VLAN10** |
| Local port | `1194` |
| Description | `SOC-Lab VPN Server (VLAN10 -> VLAN20)` |
 
**Why "SSL/TLS + User Auth":** defense in depth. Even if a user's password leaks, the attacker still needs the user's certificate. Even if a laptop with a certificate is stolen, the attacker still needs the password. Single-factor auth (cert-only or password-only) is one boundary; two-factor at the VPN edge is the entry-level expectation in any decent enterprise.
 
**Why UDP:** OpenVPN was designed for UDP. TCP-over-TCP creates retransmission stacking that degrades performance. UDP also makes the VPN harder to fingerprint with simple port scans.
 
#### Cryptographic Settings
 
| Field | Value |
| --- | --- |
| TLS Configuration | ✓ Use a TLS Key |
| Generate TLS Key | ✓ (auto on save) |
| TLS Key Usage Mode | TLS Authentication |
| Peer Certificate Authority | SOC-Lab-CA |
| Server Certificate | SOC-Lab-VPN-Server |
| DH Parameter Length | 2048 |
| Data Encryption Algorithms | AES-256-GCM, AES-128-GCM, CHACHA20-POLY1305 (defaults) |
| Fallback Data Encryption Algorithm | AES-256-CBC |
| Auth digest algorithm | SHA256 |
 
#### Tunnel Settings
 
| Field | Value |
| --- | --- |
| IPv4 Tunnel Network | `10.10.50.0/24` |
| IPv6 Tunnel Network | (blank) |
| Redirect IPv4 Gateway | **unchecked** |
| IPv4 Local Network(s) | `10.10.20.0/24` |
| Concurrent connections | `5` |
| Allow Compression | Refuse any non-stub compression (Most secure) |
| Push Compression | unchecked |
| Inter-client communication | unchecked |
| Duplicate Connections | unchecked |
 
**Why no `Redirect IPv4 Gateway`:** if checked, the client routes ALL its traffic through the VPN, breaking its own Internet access (since pfSense doesn't forward arbitrary client traffic to the WAN by default). I only need clients to reach VLAN 20, so I push only that route.
 
**Why compression off:** VORACLE and similar attacks against compressed VPN traffic. Modern best practice is no compression on VPN data planes.
 
#### Client Settings
 
| Field | Value |
| --- | --- |
| Dynamic IP | ✓ |
| Topology | Subnet |
| DNS Default Domain | `soclab.internal` |
| DNS Server 1 | (blank for now — will point to AD DC `10.10.10.10` once Phase 1.2 brings the domain online) |
 
Save. pfSense auto-generates the TLS key on save.
 
After save, **VPN → OpenVPN → Servers** shows the server with status "Up" (green).
 
### 7.5 Create user with client certificate
 
**System → User Manager → Users → Add.**
 
| Field | Value |
| --- | --- |
| Username | `vpn-corp-user` |
| Password | (strong, recorded somewhere safe) |
| Full name | Corporate VPN User |
| Group membership | (none for now) |
 
In the same form, check **Click to create a user certificate**:
 
| Field | Value |
| --- | --- |
| Descriptive name | `vpn-corp-user-cert` |
| Certificate authority | SOC-Lab-CA |
| Key Type | RSA, 2048 |
| Digest Algorithm | SHA256 |
| Lifetime (days) | 3650 |
| Common Name | `vpn-corp-user` (auto from username) |
| Certificate Type | User Certificate (default) |
 
Save.
 
### 7.6 Firewall rule on the OpenVPN tab
 
Creating the OpenVPN server makes a new tab appear: **Firewall → Rules → OpenVPN**. It's empty by default — meaning tunnel clients can connect but can't reach anything. I add the rule that allows traffic from the tunnel subnet to VLAN 20:
 
| Field | Value |
| --- | --- |
| Action | Pass |
| Interface | OpenVPN |
| Address Family | IPv4 |
| Protocol | any |
| Source | Network → `10.10.50.0/24` |
| Destination | Network → `10.10.20.0/24` |
| Description | `Allow OpenVPN clients to reach VLAN20 (dev)` |
 
Save → Apply Changes.
 
**Why `any` protocol and not just RDP/SSH:** in production I'd restrict to specific services. In a lab where I'm still iterating, I want to be able to ping VLAN20 hosts for debugging, run arbitrary tests, etc. When Phase 4 attack scenarios are designed, I'll narrow this rule to specific allowed flows — and that tightening becomes a documented decision in the scenario writeup, not silent over-permissiveness.
 
### 7.7 Export the client configuration
 
**VPN → OpenVPN → Client Export.**
 
Scroll down to the **OpenVPN Clients** table. `vpn-corp-user` should be listed (a user only appears here if it has a certificate signed by the same CA as the server's).
 
Under **Inline Configurations**, click **Most Clients** to download a `.ovpn` file with the CA, the client certificate, the client key, the TLS auth key, and the connection configuration all embedded as a single file.
 
Save this `.ovpn` somewhere I'll find later (e.g., `~/HomeLab/vpn-configs/vpn-corp-user.ovpn`). When Win11-Corp is built in Phase 1.2, this is the only file I need to import in OpenVPN Connect to bring up the tunnel.
 
---
 
## 8. Verification
 
Before snapshotting, I confirm:
 
- **Status → Interfaces:** WAN, VLAN10, VLAN20, VLAN66, VLAN99, MGMT all show "up" with correct IPs.
- **Services → DHCP Server:** four LAN ranges configured (MGMT disabled).
- **Firewall → NAT → Outbound:** mode is Automatic.
- **Firewall → Rules → MGMT:** my admin rule from §4.4 is present.
- **Firewall → Rules → OpenVPN:** the §7.6 rule is present.
- **VPN → OpenVPN → Servers:** server status "Up" (green).
- **Status → OpenVPN:** the daemon shows "running".
- **System → Cert Manager:** CA (`SOC-Lab-CA`) plus 2 certificates (`SOC-Lab-VPN-Server` + `vpn-corp-user-cert`) all valid.
- `ping 8.8.8.8` from the console works (NAT outbound).
- `https://192.168.56.10` loads with `pfctl` enabled (the MGMT rule is doing its job).
The end-to-end OpenVPN tunnel test happens in **Phase 1.2** when Win11-Corp comes online and imports the `.ovpn` — there's no way to verify tunnel routing without a real client.
 
---
 
## 9. Snapshot
 
VirtualBox → pfSense → **Snapshots → Take Snapshot**:
 
- Name: `pfsense-vpn-ready`
- Description: *pfSense 2.8 base + 4 VLAN gateways + DHCP per LAN + MGMT host-only + OpenVPN server (SSL/TLS+UserAuth, UDP 1194 on VLAN10) + CA + server cert + 1 user with cert + firewall rules on MGMT and OpenVPN. .ovpn exported. Ready for VLAN 10 endpoints in Phase 1.2.*
Rollback point before any of the next-phase VMs touches pfSense.
 
---
 
## Gotchas I hit and how I fixed them
 
These are the issues I actually encountered during this phase. Documenting them is the difference between a tutorial and a portfolio piece — they show what happens to real builds, not idealized walkthroughs.
 
### Kernel panic after first install (`vm_fault: pager read error`)
 
The first install completed but the VM booted into a kernel panic loop with `init` failing repeatedly. Cause: the install didn't write a complete bootable system to disk, likely because the ISO was removed too early during the reboot phase. Fix: hard power off, re-mount ISO, reinstall, and this time choose **Halt** at the end (not Reboot) so I can unmount the ISO manually with the VM in a clean powered-off state before booting from disk.
 
### VirtualBox GUI only shows 4 NIC tabs
 
A real engine limit on the GUI, not on the hypervisor. Slots 5–8 require `VBoxManage modifyvm --nicN ...` from the host CLI with the VM powered off. Once configured via CLI, the NIC works exactly as if it had been added in the GUI.
 
### Pre-install LAN wizard with `192.168.1.x` defaults
 
pfSense 2.8's installer added a LAN Network Mode Setup screen that pre-configures LAN before the actual disk install. The defaults are `192.168.1.1/24` with DHCP `192.168.1.100–199`. If I miss this screen and accept defaults, I end up with the wrong subnet on LAN and have to fix it via option 2 from the console post-install. Use the keyboard shortcuts (`I`, `S`, `E`) to edit IP and DHCP range before clicking Continue.
 
### Interface assignment skipped OPT prompts
 
On the very first boot after install, pfSense asked me about WAN, LAN, and then the Optional interfaces in sequence. When the second install asked for "Optional 1" right after LAN, I pressed Enter on a blank line by reflex and pfSense interpreted that as "no more interfaces" — ending up with only WAN + LAN assigned, leaving `em2`, `em3`, `em4` (and later `em5`) unassigned. Fix: re-run option `1) Assign Interfaces` from the main menu, walk through the full sequence including all OPTs, and only press Enter on the very last one to stop adding.
 
### "Should VLANs be set up now?" trap
 
Answering `y` to this question by mistake (instead of `n`) puts me inside an 802.1Q VLAN configuration sub-menu. The escape isn't obvious: at the prompt `Enter the parent interface name for the new VLAN (or nothing if finished):`, **press Enter with no input**. That exits the sub-menu back to the WAN/LAN flow. If I've already entered a parent interface and the next prompt asks for a VLAN tag, I either Ctrl+C to abort, or complete a dummy VLAN (any tag, any description) and Enter-blank on the next iteration. Any phantom VLAN I create can be deleted later from Interfaces → Assignments → VLANs tab in the GUI.
 
### Default-deny on every interface except LAN
 
When I added OPT4 (MGMT) and assigned it `192.168.56.10`, pfSense told me the Web UI was now at `https://192.168.56.10` — but my browser couldn't reach it. Cause: only LAN ships with an automatic "anti-lockout" rule; every other interface, including freshly-created OPT4, default-denies all inbound traffic. The Web UI is reachable network-wise but the firewall drops the connection. Fix: temporarily disable the packet filter with `pfctl -d` from the console shell (option 8), access the GUI, create a permanent allow rule on the MGMT tab, then re-enable with `pfctl -e`.
 
### `pfctl -d` doesn't persist
 
I initially thought I could rely on `pfctl -d` for ongoing access. Wrong — pfSense auto-re-enables the packet filter on `Apply Changes`, service restarts, and other events. Within minutes of using `pfctl -d`, normal pfSense operations turn it back on and I'm locked out again. The only stable fix is an explicit firewall rule allowing my host PC into the MGMT interface.
 
### Apply Changes drops the browser session during interface rename
 
Renaming each interface and clicking Apply Changes individually causes a `pfctl` reload after each apply. Each reload can drop my live HTTPS session with the GUI. Doing 5 renames this way = 5 chances to lose access. Fix: Save each rename without clicking Apply — the yellow "must apply" banner accumulates across pages — then click Apply Changes once at the very end. One reload, one risk window.
 
### `.local` TLD conflicts with mDNS
 
pfSense itself warns in the Setup Wizard against using `.local` as the TLD because mDNS (Avahi, Bonjour, Airprint, Windows Network Discovery in some configurations) claims it. I had originally typed `soclab.local`; the fix is `soclab.internal` — `.internal` is the IETF / ICANN convention for private networks. Change in **System → General Setup → Domain**.
 
### "Override DNS" checkbox semantics
 
The label "Allow DNS servers to be overridden by DHCP/PPP on WAN" had me checked when I wanted my manual DNS values to be sticky. Re-reading: **checked = WAN DHCP can override** (DHCP wins, my manual entries get replaced); **unchecked = manual entries are sticky** (what I actually want). For a lab where I want predictable DNS behavior, uncheck.
 
### "Block private networks" and "Block bogon networks" on WAN sabotage VirtualBox NAT
 
These two checkboxes are best practice on a public WAN — they drop spoofed packets from RFC1918 or unassigned address space. On a VirtualBox NAT WAN where my pfSense WAN IP is itself `10.0.2.15/24` (RFC1918), these rules drop legitimate traffic from the VirtualBox NAT gateway (`10.0.2.2`), including periodic DHCP renewals. Result: WAN connectivity loss without warning. Uncheck both for any virtualized NAT WAN setup.
 
---
 
## What's next
 
[`02-vlan10.md`](02-vlan10.md) — VLAN 10 build-out: Windows Server 2022 with Active Directory Domain Services, Win11-Corp domain-joined workstation. After 02 is done, Win11-Corp imports the `.ovpn` exported in §7.7, the tunnel comes up end-to-end, and I can clean up or restrict the temporary MGMT host-only interface.
 
