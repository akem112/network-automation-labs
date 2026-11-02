# FortiGate Practice Lab — 6 short labs (about 1.5 hours each)
*Goal: be able to take over an existing FortiGate environment: document it, read policies, troubleshoot, and handle basic changes. Topology: `fortigate_lab_topology.drawio`.*

## Setup (before Lab 1, about 45 min)
- **Separate EVE-NG lab.** Stop the main 12-router lab first. Each FortiGate-VM needs about 2 GB RAM (your EVE VM has 12 GB).
- **Nodes:** FW1 and FW2 (FortiGate-VM), ISP (IOSv), PC1, WEB, PC2 (Linux hosts so you can use `curl`; WEB runs a simple web server, for example `python3 -m http.server 80`).
- **Connections:** FW1 port1 and FW2 port1 to the management cloud (GUI/SSH from your PC). FW1: port2 to ISP Gi0/1, port3 to PC1, port4 to WEB. FW2: port2 to ISP Gi0/2, port3 to PC2.
- **ISP router:** Gi0/1 198.51.100.1/30, Gi0/2 198.51.100.5/30, Lo0 8.8.8.8/32 (the "internet"). No routes to the 10.x networks, so NAT is required.
- **License:** a FortiGate-VM runs on an evaluation license and has limits. Check Fortinet's current trial terms before relying on a feature.
- **FortiAP / wireless can't be simulated.** For the AP project, read the wireless controller menus (SSIDs, FortiAP profiles, VLANs) in the GUI and Fortinet's docs.

## Lab 1 — Basics: interfaces, route, outbound NAT
1. Log in, set the admin password, check version (`get system status`), take a **config backup first** (System > Config > Backup).
2. FW1: port2 198.51.100.2/30 (WAN), port3 10.1.1.1/24 (LAN), port4 10.1.99.1/24 (DMZ). Allow ping/HTTPS/SSH only on management interfaces.
3. Static default route to 198.51.100.1.
4. Policy: LAN (port3) to WAN (port2), all/ALL, **NAT enabled**, logging on.
**Verify:** PC1 pings 8.8.8.8; `get router info routing-table all`; Log & Report shows the traffic; policy hit count goes up.
**Break/fix:** remove the default route, then NAT; read the symptoms each time.

## Lab 2 — Security profiles and logs
1. Add a web filter profile that blocks a category and an application control profile that blocks one application; attach them to the LAN-to-WAN policy.
2. Use certificate inspection first; read about full SSL inspection and why it needs a trusted CA on clients.
**Verify:** blocked page shows; the log shows the profile that acted.

## Lab 3 — Objects, policy order, VIP
1. Create address objects (LAN subnet, WEB server) and a custom service (TCP 8080).
2. Order: a specific deny above a general allow; see the **implicit deny** at the bottom.
3. VIP (destination NAT) from FW1's WAN address to WEB (10.1.99.10), plus the inbound policy to the VIP.
**Verify:** from ISP (or PC2 later) `curl http://198.51.100.2`; hit counts show which policy matched.
**Break/fix:** put the allow above the deny; forget the VIP's policy.

## Lab 4 — Site-to-site IPsec (FW1 to FW2)
1. FW2: port2 198.51.100.6/30, port3 10.2.2.1/24, default route to 198.51.100.5.
2. IPsec tunnel (IKEv2 or IKEv1, pre-shared key, matching phase 1 and phase 2 settings and **selectors** 10.1.1.0/24 <-> 10.2.2.0/24).
3. Policies in both directions on both firewalls, no NAT between the LANs, static routes to the remote LAN through the tunnel interface.
**Verify:** `diagnose vpn ike gateway list`, `diagnose vpn tunnel list`, PC1 pings PC2.
**Break/fix:** wrong PSK, mismatched proposal, mismatched selectors, missing route, missing policy. Each fails differently.

## Lab 5 — Taking over a firewall
1. Back up the config (GUI, or `execute backup config tftp ...`), note firmware version and licenses.
2. Export or list every policy; flag unused (zero hit count), overly broad (all/all) and undocumented rules.
3. Admin hygiene: separate admin profiles, trusted hosts for admin access, no shared accounts.
4. Write a one-page document: interfaces, zones, VPNs, policies by purpose, contacts, contract and license dates.
**Verify:** you could hand this page to someone and they'd understand the firewall.

## Lab 6 — Troubleshooting drills (the most useful)
Make a change, close your eyes, have a fault found by method. Toolkit: `execute ping`, `get router info routing-table all`, `diagnose sniffer packet any 'host 10.1.1.10' 4`, `diagnose sys session list`, and the flow debug: `diagnose debug flow filter addr 10.1.1.10`, `diagnose debug flow trace start 10`, `diagnose debug enable` (stop with `diagnose debug reset`; syntax varies by FortiOS version).
Ask in order: interface up? route? policy match? NAT? VPN up? Faults to try: wrong policy order, NAT off, missing route, wrong subnet on an interface, shut interface, VPN selector mismatch.

## Optional, if time/memory allows
SD-WAN with two WAN links (performance SLA), HA between two FortiGates, FortiManager/FortiAnalyzer concepts (read only).

## Schedule suggestion (1–2 hours a day, alongside ENARSI)
| Day | Work |
|---|---|
| 1 | Setup + Lab 1 |
| 2 | Lab 2 |
| 3 | Lab 3 |
| 4 | Lab 4 |
| 5 | Lab 5 |
| 6 | Lab 6 |
| 7 | Review, rest, back to ENARSI |

## Exam link
Fortinet's NSE 4 (FortiGate Security and Infrastructure) covers these topics: policies and NAT, security profiles, IPsec VPN, SD-WAN, HA and routing. Check Fortinet's current exam guide for the exact objectives.
