# Network automation labs (EVE-NG + Ansible)

Hands-on routing, security and automation labs built while studying for CCNP ENARSI (300-410).
Everything is lab work in EVE-NG: Cisco IOSv routers configured with Ansible, plus a separate
FortiGate practice lab and an SD-WAN design.

## What's here
| Path | What it is |
|---|---|
| `playbooks/` | Ansible playbooks for the 12-router ENARSI lab (one switch variable per lab, a full reset before each run) |
| `docs/enarsi-lab-notes.md` | Topology, redistribution and loop-prevention design, lessons learned, progress |
| `docs/fortigate-lab-plan.md` | Six short FortiGate labs: NAT, security profiles, VIP, IPsec, taking over a firewall, troubleshooting |
| `docs/sdwan-lab-plan.md` | Dual-ISP FortiGate SD-WAN lab with Cisco routing and automation |
| `diagrams/` | draw.io topologies (open in draw.io) |
| `inventory_example.ini`, `ansible.cfg.example` | Templates. The real inventory and config are git-ignored |

## The ENARSI lab
12 IOSv routers (R1 to R12): EIGRP, OSPF, eBGP, a DMVPN Phase 3 hub-and-spoke with IPsec, and
two-way redistribution with route-maps and tags to prevent loops.

| Playbook | Lab switch |
|---|---|
| `configure_ospf_area_labs.yml` | `-e lab=baseline\|totally_stubby\|nssa\|virtual_link\|filter_list\|distribute_list` |
| `configure_pbr_ipsla.yml` | `-e pbr_lab=off\|pbr\|pbr_track\|static_track` |
| `configure_ibgp_rr.yml` | `-e bgp_lab=off\|full_mesh\|hub_only\|rr\|rr_community` |
| `configure_bgp_paths.yml` | `-e path_lab=off\|baseline\|weight\|local_pref\|prepend\|med\|order` |
| `configure_mgmt_static.yml` | `-e apply=true` (plan first, then apply) |
| `break_lab.yml` / `restore_lab.yml` | Inject random faults and restore (troubleshooting practice) |

Each lab playbook removes the previous lab's config first, then applies the one you name, so labs can
be run in any order.

## Running it
1. Copy `inventory_example.ini` to `inventory_enarsi.ini`, set real addresses, and `export LAB_PASSWORD=...`.
2. Copy `ansible.cfg.example` to `ansible.cfg`.
3. **Check the inventory before every run** (addresses can shift after a restart):
   `ansible -i inventory_enarsi.ini all -m cisco.ios.ios_command -a "commands='show run | include hostname'"`
4. Export the keys the IPsec and EIGRP-auth playbooks read: `export DMVPN_PSK=...` and `export EIGRP_KEY=...`.
5. Run a playbook, for example `ansible-playbook -i inventory_enarsi.ini playbooks/configure_ibgp_rr.yml -e bgp_lab=rr`.

Requires Ansible with the `cisco.ios` and `ansible.netcommon` collections.

## Secrets
No credentials are stored in this repo. The inventory password, the DMVPN pre-shared key and the EIGRP
key come from environment variables (or Ansible Vault). Never commit your real inventory.
