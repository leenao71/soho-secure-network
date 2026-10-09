# Threat Model

## Scope and assumptions

**Scope:** a small office network with five VLANs, a DMZ web server, one router, two switches, a central syslog server and a simulated internet.

**Assets:** staff data and workstations, the Internal Server, the DMZ web server, device admin access and the audit logs.

**Attackers considered:**
1. An outsider on the internet
2. A malicious or careless guest
3. A compromised IoT device
4. A compromised DMZ web server
5. Someone physically plugging a device into a spare port

**Out of scope:** malware on Staff PCs, phishing, insider theft by an admin, and physical theft of devices.

## Trust zones

| Zone | VLAN / subnet | Trust | Can start connections to |
|---|---|---|---|
| Staff | 10, 192.168.10.0/24 | High | Everything |
| Guest | 20, 192.168.20.0/24 | None | Internet only |
| IoT | 30, 192.168.30.0/24 | Low | Internet only |
| Servers | 40, 192.168.40.0/24 | High | Internet (replies only inside) |
| DMZ | 50, 172.16.50.0/24 | Low (exposed) | Nothing inside (replies only) |
| Management | 99, 192.168.99.0/24 | High | Not NATed to the internet |

## Threat table

| # | Asset | Threat | Attack example | Control that stops it | Evidence |
|---|---|---|---|---|---|
| 1 | Network segmentation | VLAN hopping, flat-network lateral movement | DTP trunk negotiation; double-tagged frames using the native VLAN | Separate VLANs per trust zone; static access/trunk ports; DTP disabled; unused native VLAN 999; trunk allowed-VLAN list | 04, 05 |
| 2 | Internal hosts and public web server | Inbound attack or reconnaissance from the internet | Attacker scans the public address and tries to reach Staff PCs or the Internal Server | PAT with no inbound mappings except one static NAT; DMZ in its own VLAN; management VLAN not NATed | 11, 12, 12b, 14 |
| 3 | Switches and router | Unauthorised admin access, sniffed credentials, rogue devices | Telnet password sniffing; password guessing; laptop plugged into a spare port | SSH v2 only; Telnet refused; hashed secrets; login banner; port security (sticky, max 1, shutdown); unused ports shut down in VLAN 999 | 15, 16-17, 18, 19, 20, 21, 22 |
| 4 | Staff and Server networks | Lateral movement from a compromised Guest, IoT or DMZ device | Hacked camera scans the Servers VLAN; hacked web server pivots inward | Inbound extended ACLs per VLAN; DMZ may only reply; Guest and IoT blocked from management | 03-BEFORE, 03-AFTER, 33 |
| 5 | Audit trail | Attacks go unnoticed, or logs are erased | Repeated SSH password guessing; rogue laptop on a spare port | Central syslog server in the Servers VLAN; login logging; port-security violation messages; Guest and IoT cannot reach the log server | 35, 36, 38, 39, 40, 41, 42 |

## Residual risks

Even with these controls, an attacker who compromises a Staff PC has the widest access, because Staff is trusted and can reach Servers, IoT and management. The ACLs are stateless, so the `established` keyword trusts any packet that looks like a reply. The DMZ shares the router's physical trunk with the other VLANs, so one misconfiguration could expose it. The syslog server uses unauthenticated UDP, so a forged message could mislead an analyst.

## Limitations

| Limitation | Why it matters |
|---|---|
| No real firewall. R1 is a router with ACLs. | ACLs are stateless and do not inspect applications. A real firewall tracks connections. |
| No IDS/IPS. Nothing detects attacks. | Logs are collected, but nothing analyses or alerts on them. |
| DMZ is a VLAN on a router sub-interface, not its own physical port. | A real DMZ sits on a dedicated firewall interface. |
| SSH source addresses were not logged. Packet Tracer records SSH logins as `console` with `Source: 0.0.0.0`. | Remote brute-force detection could not be demonstrated. |
| No login lockout on the switches. `login block-for` was rejected there. | The switches rely on the SSH retry limit only. |
| Manual clocks. Time resets on reload, and the syslog server stamps entries with its own clock. | Real logs need NTP for trustworthy timestamps. |
| 1024-bit RSA key and type 7 `service password-encryption`. | Modern guidance is 2048-bit or larger, and type 7 is trivially reversible. Only the `secret` hashes are strong. |
| Local accounts with a shared username. | No per-person accounts or audit by name. A real network uses TACACS+ or RADIUS. |
| Syslog is UDP and unencrypted. | Messages can be forged or intercepted. |
| IPv6 not configured or secured. | A real network must cover it. |
| ACL hits are not logged; evidence is `show access-lists` counters. | You can prove what matched, but there is no event stream for it. |
| Single point of failure: one router, one trunk. | No redundancy. |

## Next steps

| Step | What it adds |
|---|---|
| Rebuild the firewall layer in pfSense, with the DMZ on its own interface | Stateful filtering and a proper DMZ |
| Add Wazuh (free SIEM) and forward syslog to it | Alerts on brute force, port violations and config changes |
| NTP on every device | Trustworthy timestamps across logs |
| TACACS+ or RADIUS for admin logins | Per-user accounts and accountability |
| DHCP snooping, Dynamic ARP Inspection, BPDU guard | Stops rogue DHCP servers and ARP spoofing |
| 802.1X port authentication | Authenticates devices before they get network access |
| 2048-bit RSA keys and SSH key authentication | Stronger admin access |
| Add IPv6 and secure it | Closes a common blind spot |
| Automate the build with Ansible or Python (Netmiko) | Repeatable, reviewable configs |
