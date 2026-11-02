# ENARSI main lab: notes


## Goal
Practice for CCNP ENARSI (300-410) in an EVE-NG lab configured with Ansible.

## Topology (12 IOSv routers, er1–er12)
| Part | Routers | Notes |
|---|---|---|
| EIGRP AS 100 (named mode `EIGRP-LAB`) | R1, R2, R3 triangle, plus R4 and R6 | R1 Gi0/3 ↔ R6 (10.10.20.0/30) |
| OSPF | R4–R5–R6 area 0, R6–R7 area 1 (stub), R5–R8 area 0 | |
| eBGP | R8 AS65001 ↔ R9 AS65002 (transit) ↔ R10 AS65003 | |
| DMVPN Phase 3 | R10 hub, R11 and R12 spokes, EIGRP AS 200 (`DMVPN`) | Tunnel0 172.16.100.0/24: R10 .1, R11 .11, R12 .12 |

Links: R1–R2 10.10.1.0/30, R2–R3 10.10.2.0/30, R1–R3 10.10.3.0/30, R3–R4 10.10.4.0/30, R4–R5 10.10.5.0/30, R5–R6 10.10.6.0/30, R6–R7 10.10.8.0/30, R5–R8 10.10.9.0/30, R8–R9 10.10.10.0/30, R9–R10 10.10.11.0/30, R10–R11 10.10.12.0/30, R10–R12 10.10.13.0/30. Loopbacks 10.255.0.x/32. Test prefixes: R9 10.99.9.0/24, R10 10.100.10.0/24, R11 10.111.1.0/24, R12 10.112.1.0/24.

## Redistribution and loop prevention (tags)
- **R4 and R6 (EIGRP 100 ↔ OSPF):** `EIGRP-TO-OSPF` deny 5 match tag 200, permit 10 set tag 100. `OSPF-TO-EIGRP` deny 5 match tag 100, permit 10 set tag 200. (R4's deny 5 on `OSPF-TO-EIGRP` was added on 2026-10-12.)
- **R8 (OSPF ↔ BGP):** `BGP-TO-OSPF` sets tag 300; `OSPF-TO-BGP` denies tag 300, permits the rest. OSPF into BGP uses `match internal external 1 external 2`.
- **R10 (EIGRP 200 ↔ BGP):** only the spoke LANs go into BGP (prefix-list `SPOKE-LANS`); BGP into EIGRP 200 is tagged 500 with the spoke LANs denied. The hub sends spokes only a default summary.
- End-to-end tests pass: R1 ↔ R11 pings between loopbacks and LANs.

## Ansible (in ~/ansible-lab on the Ansible VM)
- Inventory `inventory_enarsi.ini`: groups `eigrp_lab` (also holds er5–er7 for connection vars), `ospf_lab`, `bgp_lab` (er8–er12, own `:vars`). Management IPs are on 10.0.0.0/24 (DHCP; check them if a router "has no route").
- Playbooks: `configure_interfaces.yml`, `configure_eigrp.yml` (has `when: eigrp_networks is defined`), `configure_ospf.yml`, `configure_redistribution.yml` (R4/R6 tag design, saves config), `configure_eigrp_auth.yml` (written, NOT run yet), `configure_eigrp_summary.yml` (written, not confirmed run), `configure_ospf_area_labs.yml` (switch: `-e lab=baseline|totally_stubby|nssa|virtual_link|filter_list|distribute_list`), `configure_pbr_ipsla.yml` (switch: `-e pbr_lab=off|pbr|pbr_track|static_track`, plus `-e pbr_next_hop=10.10.3.2 -e pbr_sla_source_if=GigabitEthernet0/2` for the R3 path), `configure_dmvpn_ipsec.yml`, `configure_r8_bgp.yml`, `configure_r10_bgp.yml`, `configure_dmvpn.yml`, `configure_r10_redistribution.yml`, `break_lab.yml`, `restore_lab.yml`.
- `break_lab.yml` injects 4 of 12 faults and writes the hidden `answer_key.txt`. `restore_lab.yml` reverts all 12, then re-runs the configure playbooks including `configure_redistribution.yml`. Both are still scoped to R1–R7 only; not yet run live as a full break/restore cycle.
- Fault pool (12): eigrp_wrong_as, eigrp_passive_interface, eigrp_remove_network, ospf_wrong_area, ospf_hello_mismatch, ospf_wrong_wildcard, ospf_interface_down, redist_remove_subnets, redist_wrong_tag, redist_remove_routemap, stub_mismatch, loopback_wrong_mask.

## Lessons learned
- **Save configs.** A node restart in EVE-NG wipes unsaved config. New playbooks use `save_when: modified`. Run `write memory` before stopping for the day.
- In EIGRP named mode, `passive-interface` goes under `af-interface <intf>`, not directly under the address-family (fixed in break and restore).
- New routers need an SSH key accepted on the Ansible VM first (`ssh-keyscan -t rsa <ip> >> ~/.ssh/known_hosts`) and the `network_cli` connection vars in the inventory.
- EVE-NG VM has 12 GB; 9 nodes used about 69% memory. With 12 nodes, stop idle nodes (R1–R3, R7) or raise to 14 GB. R10 uses all 4 IOSv interfaces; a fifth needs more Ethernets on that node.
- **IP SLA:** `timeout` cannot be lower than `threshold` (default 5000 ms), so set `threshold` first. Pin a next-hop probe to the exit interface with `source-interface`; otherwise it can reach the next hop the long way round and the track stays Up while the link is down.
- **PBR:** `set ip next-hop` needs an adjacent next hop; if it is only reachable through another router the policy is skipped silently. `ip local policy` only affects traffic R1 originates. The match counter in `show route-map` is the first thing to check.
- **OSPF:** a loopback is advertised as a /32 regardless of its mask (check `10.77.8.1`, not `10.77.8.0`). A transit area for a virtual link cannot be stub or NSSA. In an NSSA, R6 (an ASBR) floods its own redistributed routes into area 1 as Type 7.
- **Ansible:** a flag typed without a space before it (`pbr--cheeck`) becomes part of the variable value; `no ...` reset tasks always report "changed", and the red "Operation not found" on removing a missing probe is ignored by design.
- **Inventory and DHCP (important):** router management addresses are DHCP and **shuffled after the Oct 28 node restart**, so inventory names no longer matched routers and a BGP playbook configured R7 instead of R3. Always run `ansible -i inventory_enarsi.ini all -m cisco.ios.ios_command -a "commands='show run | include hostname'"` after a lab restart and confirm erN returns RN before running any playbook. Fixed on Oct 28 (backup: `inventory_enarsi.ini.good`). Permanent fix still to do: static management addresses on all 12 routers. Do not run `restore_lab.yml` or `break_lab.yml` until the map is verified.
- **Ansible and IOSv:** stale SSH host keys cause "host key has changed"; for this lab `host_key_checking = False` in `ansible.cfg` avoids it. IOSv is slow after a BGP reset: a 30 s command timeout on all three routers means set `command_timeout = 90` under `[persistent_connection]` and rerun (the playbooks reset first, so reruns are safe). `show ... | include` is case sensitive (`Reflector`, not `reflector`).
- Route AD protects R4 from re-injecting BGP routes today (OSPF AD 110 beats EIGRP 170); tags are the safety net for failures.

## Domain 1 (Layer 3 Technologies) progress
| Module | Status |
|---|---|
| 1 EIGRP | Feasible successor lab done. Authentication (`configure_eigrp_auth.yml`) and summarization (`configure_eigrp_summary.yml`) written, results not reported yet. |
| 2 OSPF advanced | Done and verified: totally stubby, NSSA (Type 7 to Type 5 translation seen on R5), virtual link (R7 area 2 via area 1), filter-list vs distribute-list (LSA stays in the database with distribute-list). |
| 4 PBR and IP SLA | Done and verified: local PBR to the R3 path, tracked PBR (`verify-availability`), failover to R6 and recovery. `static_track` (tracked static route) not run yet. |
| 5 BGP, part 1: iBGP, route reflectors, communities | **Done and verified (Oct 29)** with `configure_ibgp_rr.yml` (R1, R2, R3; AS 65100; switch `-e bgp_lab=off\|full_mesh\|hub_only\|rr\|rr_community`). full_mesh: 3 sessions, each peer sends only its own prefix. hub_only: R2 lacks 10.20.3.0/24 and R3 lacks 10.20.2.0/24 (iBGP split horizon). rr: R1 reflects; R3 sees Originator 10.255.0.2 and Cluster list 10.255.0.1, next hop unchanged. rr_community: R2 tags 65100:222, R3 denies it inbound from R1. Set back with `bgp_lab=off`. |
| 5 BGP, part 2: path selection (weight, local preference, AS-path prepend, MED) | Not started. Needs a new R8 Gi0/3 to R9 Gi0/3 link in EVE for a second path. |
| Modules 6, 9, 10 (later domains) | Not started. |

## Practice so far
- K-value mismatch (solved); loop prevention (answered, corrected: tag + route-map, not removing the link).
- Still open: "OSPF into EIGRP with no metric" scenario (EIGRP needs a metric from a non-EIGRP source; two fixes: `metric` on the redistribute line, or `default-metric`).

## Next steps (pick one)
1. Run `configure_eigrp_auth.yml` (MD5 with key chain; skips R1–R6 link) and `configure_eigrp_summary.yml`, and report results.
2. Run `configure_pbr_ipsla.yml -e pbr_lab=static_track` and test failover, then set `pbr_lab=off`.
3. Module 5 part 2: BGP path selection (weight, local preference, AS-path prepend, MED, best-path order). Add the R8 Gi0/3 to R9 Gi0/3 link first. (Part 1, route reflectors and communities, is done.)
3b. First: set static management addresses on all 12 routers so the inventory stays correct across restarts.
4. (IPsec on the DMVPN tunnel is done: IKEv1 PSK, QM_IDLE, UP-ACTIVE.) Extend `break_lab.yml` / `restore_lab.yml` to BGP and DMVPN faults, then run a full break → troubleshoot → restore cycle.
5. Come back to the open redistribution questions and mock troubleshooting rounds.
