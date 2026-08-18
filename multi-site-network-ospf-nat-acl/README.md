# Multi-Site Routed & Switched Network — Cisco Packet Tracer Lab

A self-contained enterprise LAN/WAN lab built in Cisco Packet Tracer, covering VLAN
segmentation, router-on-a-stick inter-VLAN routing, single-area OSPF with DR/BDR
election, centralized DHCP with relay, NAT overload (PAT) to a simulated ISP, and
extended/standard ACLs for both traffic filtering and management-plane security.

![Topology](diagrams/topology.svg)

## Goal of this lab

Design and build, from scratch, a small multi-site enterprise network that a real
business could plausibly run: two office sites each with segmented departments,
a central HQ providing shared services and internet breakout, and the routing,
addressing, and access-control policy needed to make it all work correctly and
securely — not just "pingable," but built the way it would actually be designed
(segmented broadcast domains, a routing protocol instead of static routes at
scale, centralized services reached across subnets, controlled internet egress,
and least-privilege access between departments/sites).

## Skills demonstrated

- **VLAN design & 802.1Q trunking** — segmenting departments into separate
  broadcast domains and trunking multiple VLANs over a single uplink
- **Inter-VLAN routing** via router-on-a-stick (sub-interfaces, per-VLAN gateways)
- **Dynamic routing with OSPF** — single-area design, adjacency states, DR/BDR
  election on a multi-access segment, route redistribution of a default route
- **IP addressing & subnetting** — VLSM-style plan across four LANs, a core
  segment, and a WAN link
- **DHCP services** — centralized server with per-subnet scopes and DHCP relay
  (`ip helper-address`) across router boundaries
- **NAT/PAT** — many-to-one address translation for internet-bound traffic,
  paired with a standard ACL defining translation scope
- **Access control (ACLs)** — extended ACL for inter-VLAN traffic policy,
  standard ACL for securing remote management (VTY) access
- **Systematic verification** — `show` command-driven testing methodology
  rather than "it pings so it's done"

## Topology

| Device | Role |
|---|---|
| `SW-A`, `SW-B` | Access-layer switches, 802.1Q trunks to their site router |
| `R1`, `R2` | Site edge routers, router-on-a-stick inter-VLAN routing |
| `R3` | HQ/core router — OSPF, NAT overload, VTY security |
| `SW-CORE` | Shared broadcast segment joining R1/R2/R3 (forces OSPF DR/BDR election) |
| `SRV-DHCP` | Central DHCP server, one pool per VLAN |
| `R-ISP` | Simulated internet edge |

## Addressing plan

| Segment | Network | Gateway |
|---|---|---|
| VLAN 10 — Site A Sales | `10.10.10.0/24` | `.1` on R1 sub-interface |
| VLAN 20 — Site A IT | `10.10.20.0/24` | `.1` on R1 sub-interface |
| VLAN 30 — Site B Sales | `10.20.30.0/24` | `.1` on R2 sub-interface |
| VLAN 40 — Site B IT | `10.20.40.0/24` | `.1` on R2 sub-interface |
| Core (R1/R2/R3/Server) | `10.0.0.0/29` | R1 `.1` · R2 `.2` · R3 `.3` · Server `.4` |
| WAN (R3 ↔ R-ISP) | `203.0.113.0/30` | R3 `.1` · R-ISP `.2` |

Router model: Cisco 1941. Switch model: Cisco 2960. Server: generic Server-PT.

## What's implemented, and why

**VLAN segmentation + 802.1Q trunking** — each site's switch separates Sales and
IT into their own broadcast domains (VLAN 10/20 at Site A, 30/40 at Site B). The
uplink to the site router is a trunk carrying an 802.1Q tag per frame so a single
physical link can serve multiple VLANs; the native VLAN is moved off the default
(VLAN 1) to VLAN 99 as a minor hardening step against native-VLAN mismatch/hopping.

**Router-on-a-stick** — R1 and R2 each terminate the trunk on one physical
interface split into per-VLAN logical sub-interfaces (`gi0/0.10`, `gi0/0.20`, ...),
each with its own IP acting as that VLAN's default gateway.

**Single-area OSPF (Area 0)** — R1, R2, and R3 all attach to a shared switched
segment (`SW-CORE`) rather than point-to-point links, specifically so OSPF's
DR/BDR election is exercised: on a multi-access segment, routers elect one
Designated Router (and a backup) to avoid an O(n²) mesh of full adjacencies.
Router IDs are set explicitly (`1.1.1.1` / `2.2.2.2` / `3.3.3.3`) for
deterministic, easily verifiable election results.

**Centralized DHCP with relay** — one DHCP server hosts four pools (one per
VLAN) rather than one server per site. Since DHCP discovery is a broadcast and
routers don't forward broadcasts across subnets, each client-facing
sub-interface on R1/R2 runs `ip helper-address 10.0.0.4` to unicast-relay the
request to the server and back.

**NAT overload (PAT) at the HQ edge** — R3 translates all four internal subnets
to its single public IP (`203.0.113.1`) using port-based overload, governed by a
standard ACL defining which sources are eligible for translation. A default
route on R3, redistributed into OSPF via `default-information originate`, gives
R1 and R2 a path for any traffic that isn't one of the known internal subnets.

**ACLs** — two demonstrations:
- *Extended ACL 120* on R1 (`gi0/0.20 in`) blocks Site A IT (VLAN 20) from
  reaching Site B Sales (VLAN 30) specifically, while explicitly permitting
  everything else — illustrating first-match, top-down ACL processing and the
  implicit deny-all that makes the trailing `permit ip any any` mandatory.
- *Standard ACL 10* applied with `access-class ... in` on R3's VTY lines
  restricts remote management access to the four internal subnets only,
  keeping Telnet reachable from inside the network but not from the WAN side.

## Repo structure

```
cisco-lan-lab/
├── README.md
├── network.pkt              ← add your Packet Tracer file here
├── screenshots/              ← add your verification screenshots here
│   ├── ospf-neighbors.png
│   ├── nat-translations.png
│   ├── dhcp-binding.png
│   └── acl-counters.png
├── diagrams/
│   └── topology.svg
└── configs/
    ├── R1.txt
    ├── R2.txt
    ├── R3.txt
    ├── R-ISP.txt
    ├── SW-A.txt
    └── SW-B.txt
```

`configs/` holds the final running-config for each device — paste-ready into a
fresh device in Packet Tracer, or usable to diff against your own build.

### Screenshots

*(Add screenshots here as you take them — recommended shots: `show ip ospf
neighbor` with DR/BDR roles visible, `show ip nat translations` mid-ping,
`show vlan brief` on both switches, `show access-lists` with match counters
incrementing, and a successful cross-VLAN ping.)*

## Verification performed

- `show vlan brief` on `SW-A` / `SW-B` — correct VLAN-to-port mapping
- `show interfaces trunk` — trunk up, correct allowed-VLAN list, native VLAN 99
- `show ip interface brief` on `R1` / `R2` — all sub-interfaces `up/up`
- `show ip ospf neighbor` / `show ip ospf interface gi0/0` — full adjacency,
  DR/BDR roles confirmed across R1/R2/R3
- End-to-end DHCP: all 8 PCs pull correct per-VLAN addressing via relay
- Cross-VLAN, cross-site ping (e.g. `PC-A1 → PC-B3`) succeeds via OSPF-learned
  routes
- `show ip nat translations` on `R3` — live inside-local ↔ inside-global
  mappings while pinging out to `R-ISP`
- `show ip route` on `R1` — `O*E2 0.0.0.0/0` present (external default learned
  via OSPF from R3)
- `show access-lists` — match counters increment on ACL 120's deny line when
  VLAN20 → VLAN30 traffic is attempted; ping from VLAN20 to VLAN10/40 still
  succeeds
- Telnet to `R3` (`10.0.0.3`) succeeds from an internal subnet, refused from
  outside the permitted ACL 10 ranges

## Possible extensions

- Multi-area OSPF as the topology grows, to bound SPF recalculation scope
- HSRP/VRRP for default-gateway redundancy at each site
- Port security on access-layer switchports
- Named ACLs for readability at scale
- EIGRP or static-route redistribution comparison against OSPF

## Author's note

Built as a hands-on lab to practice VLAN design, inter-VLAN routing, OSPF
adjacency/election behavior, DHCP relay, NAT/PAT, and ACL policy — end to end,
on Cisco IOS in Packet Tracer.
