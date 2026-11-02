# ENARSI + FortiGate SD-WAN + Automation Lab (separate EVE-NG lab)
*Goal: practice the skills network engineer postings ask for together: routing and switching, security, SD-WAN, automation. Reuses your ENARSI knowledge. Topology: `enarsi_sdwan_topology.drawio`.*
*Keep it separate from the 12-router ENARSI lab and the earlier FortiGate lab. Run only one lab at a time (EVE-NG VM has 12 GB).*

## Honest scope
- The ENARSI exam does **not** cover FortiGate or SD-WAN. This lab goes beyond it, because network roles now blend routing and switching, security, SD-WAN and automation. Use it **alongside** exam study, not instead of it.
- **Cisco Catalyst SD-WAN** (vManage/vSmart/vBond controllers) is too heavy for your EVE VM. Learn it conceptually (Cisco U, DevNet sandbox) and practice SD-WAN ideas on **FortiGate SD-WAN**, which runs on two small VMs.
- FortiGate-VM trial license: 1 vCPU, 2 GB RAM per node. Check in Lab 1 that port1 to port4 all work. Features that depend on FortiGuard (application signatures, the internet service database, web categories) won't be current, so steer traffic by address, port and SLA, not by application name.

## Nodes (about 7 GB RAM total)
| Node | Type | Role |
|---|---|---|
| FW1 | FortiGate-VM (4 Ethernets) | HQ firewall and SD-WAN hub |
| FW2 | FortiGate-VM (4 Ethernets) | Branch firewall and SD-WAN spoke |
| ISP-A, ISP-B | IOSv | Two internet providers |
| INET | IOSv | "The internet": Loopback 8.8.8.8/32 |
| CORE | IOSv | HQ core router (OSPF/EIGRP/BGP to FW1) |
| PC1, PC2 | Linux or VPCS | HQ and branch hosts (use Linux where you need `curl`) |

Management: FW port1 and every IOSv Gi0/0 to a management cloud your Ansible VM can reach (DHCP; set **static** addresses so the inventory never goes stale).

## Addressing
| Link | Subnet | Addresses |
|---|---|---|
| FW1 port2 (WAN1) to ISP-A Gi0/1 | 198.51.100.0/30 | FW1 .2, ISP-A .1 |
| FW1 port3 (WAN2) to ISP-B Gi0/1 | 203.0.113.0/30 | FW1 .2, ISP-B .1 |
| FW2 port2 (WAN1) to ISP-A Gi0/2 | 198.51.100.4/30 | FW2 .6, ISP-A .5 |
| FW2 port3 (WAN2) to ISP-B Gi0/2 | 203.0.113.4/30 | FW2 .6, ISP-B .5 |
| ISP-A Gi0/3 to INET Gi0/1 | 192.0.2.0/30 | ISP-A .1, INET .2 |
| ISP-B Gi0/3 to INET Gi0/2 | 192.0.2.4/30 | ISP-B .1, INET .6 |
| FW1 port4 to CORE Gi0/1 | 10.10.1.0/30 | FW1 .1, CORE .2 |
| HQ LAN (CORE Gi0/2, PC1) | 10.1.1.0/24 | CORE .1, PC1 .10 |
| Branch LAN (FW2 port4, PC2) | 10.2.2.0/24 | FW2 .1, PC2 .10 |
| INET Loopback0 | 8.8.8.8/32 | |

ISP-A, ISP-B and INET run static routes or a simple BGP between them so 8.8.8.8 is reachable through both ISPs. They have no routes to the 10.x networks, so NAT or an overlay is required.

## How ENARSI skills map to FortiGate
| ENARSI skill | FortiGate equivalent |
|---|---|
| IP SLA + track + tracked static route | SD-WAN **performance SLA** / **link monitor** |
| Policy-based routing | SD-WAN **rules** (and policy routes) |
| Static routes, administrative distance | Static routes with distance and priority |
| EIGRP/OSPF/BGP, redistribution, route-maps | FortiOS OSPF/BGP, route-maps, redistribution |
| ECMP, load sharing | SD-WAN load-balance strategy |
| DMVPN / IPsec | IPsec tunnels as SD-WAN members (ADVPN for dynamic tunnels) |
| BFD | FortiOS BFD |
| Syslog, SNMP, NetFlow | Log settings, SNMP, sFlow/NetFlow |

## Labs (about 2 hours each)

**Lab 1: dual-WAN and routing foundation.** Configure FW1 (port2 WAN1, port3 WAN2, port4 to CORE), ISPs and INET, default routes on both WANs, outbound NAT. Run OSPF or BGP between FW1 and CORE and advertise the HQ LAN. **Verify:** PC1 pings 8.8.8.8, `get router info routing-table all`, session list shows the NAT. **Break/fix:** two default routes with the same distance (ECMP), then different distances; DHCP-learned routes on the management port winning over your static route.

**Lab 2: SD-WAN basics.** Create an SD-WAN zone with both WANs as members. Add a **performance SLA** (ping 8.8.8.8, thresholds for latency, jitter, loss). Add SD-WAN rules: "prefer WAN1, fall back to WAN2" and "best quality". **Verify:** `diagnose sys sdwan member`, `diagnose sys sdwan service`, `diagnose sys sdwan health-check`. **Fail test:** shut ISP-A Gi0/1, watch the SLA mark WAN1 dead and the traffic move to WAN2, then restore. This is the FortiGate version of your tracked PBR lab.

**Lab 3: IPsec overlay and SD-WAN over it.** Build FW2 (branch). Create two IPsec tunnels between FW1 and FW2, one over each WAN, and put both in the SD-WAN zone. Route the LANs over the overlay (BGP or OSPF over the tunnels). **Verify:** `diagnose vpn ike gateway list`, `diagnose vpn tunnel list`, PC1 to PC2 pings, then fail one WAN and watch the traffic continue. **Break/fix:** wrong pre-shared key, mismatched proposals, selector mismatch, missing route or policy.

**Lab 4: steering and SLA tuning.** Steer by destination, port and protocol (for example, HTTP via WAN1, ICMP via WAN2). Tune SLA thresholds, hold-down and fail-back. Compare **manual**, **best quality**, **lowest cost** and **maximize bandwidth** strategies. Read the SD-WAN logs and graphs.

**Lab 5: integrate with the Cisco core.** Redistribute between OSPF/EIGRP/BGP on CORE and FW1 with tags and route-maps (your ENARSI loop-prevention design). Test summarization, route filtering and a route leak. Keep this tied to your Domain 1 study.

**Lab 6: automation and break/fix.** Use Ansible: `cisco.ios` for the IOSv routers (as you already do) and the `fortinet.fortios` collection (or plain REST calls from Python) for the FortiGates. Playbooks to write: config backup of all nodes, push interfaces and SD-WAN config, verify (SLA state, tunnel state, routes), and a break script that injects faults for you to find. Use a REST API admin token with an API user restricted to what the playbooks need, kept in Ansible Vault or an environment variable, not in the playbook.

## Break/fix drills (use in November)
Wrong gateway, missing route, SLA pointing at an unreachable target, tunnel selector mismatch, NAT disabled, policy order, overlapping subnets, redistribution loop, asymmetric routing, interface shut on one WAN, clock or certificate problems.

## How to describe this lab
"I built a lab with a dual-ISP FortiGate SD-WAN design: performance SLAs, rule-based steering, IPsec overlay between HQ and branch, failover tests, integration with Cisco routing, and Ansible for backups, deployment and verification." This is lab work, not production experience.

## Schedule suggestion
- Now to Nov 14: build the topology and finish Lab 1 and Lab 2 (about 4 hours total, two evenings).
- Nov 15 to Dec 10: one SD-WAN lab a week plus use the topology for break/fix.
- After Dec 15: Labs 3 to 6 in depth and the Fortinet NSE 4 material.
