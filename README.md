# Multi-Site-Enterprise-Network

A 3-site hub-and-spoke enterprise network built in Cisco Packet Tracer, featuring VLAN segmentation, inter-VLAN routing, GRE-over-IPsec site-to-site VPN tunnels, OSPF dynamic routing, and centralized DHCP/DNS via relay. 

## Overview

- **HQ** (hub) — 3 VLANs (Sales, Engineering, Servers), centralized DHCP/DNS server, router-on-a-stick inter-VLAN routing
- **Branch A** and **Branch B** (spokes) — one VLAN each, no local servers, fully dependent on HQ for DHCP/DNS
- Each branch connects to HQ through a simulated ISP transit router, with a dedicated GRE tunnel carrying OSPF-advertised routes between HQ and each branch
- Branches never communicate directly — all inter-site traffic transits through HQ (true hub-and-spoke)

## Topology

```
                    HQ (hub)
              3 VLANs, DHCP/DNS
                     |
                 ISP router
               (transit only)
              /              \
      Branch A                Branch B
      (spoke)                  (spoke)
```

Two GRE tunnels overlay this physical path:
- **Tunnel 1**: HQ ↔ Branch A
- **Tunnel 2**: HQ ↔ Branch B

## IP Addressing

| Segment | Subnet | Notes |
|---|---|---|
| HQ — Sales (VLAN 10) | 10.10.10.0/24 | Gateway 10.10.10.1 |
| HQ — Engineering (VLAN 20) | 10.10.20.0/24 | Gateway 10.10.20.1 |
| HQ — Servers (VLAN 99) | 10.10.99.0/24 | Gateway 10.10.99.1, server at 10.10.99.10 |
| Branch A — Users (VLAN 30) | 10.20.30.0/24 | Gateway 10.20.30.1 |
| Branch B — Users (VLAN 40) | 10.30.40.0/24 | Gateway 10.30.40.1 |
| HQ ↔ ISP (WAN) | 203.0.113.0/30 | HQ .1, ISP .2 |
| Branch A ↔ ISP (WAN) | 198.51.100.0/30 | Branch A .1, ISP .2 |
| Branch B ↔ ISP (WAN) | 192.0.2.0/30 | Branch B .1, ISP .2 |
| Tunnel 1 (HQ ↔ Branch A) | 172.16.1.0/30 | HQ .1, Branch A .2 |
| Tunnel 2 (HQ ↔ Branch B) | 172.16.2.0/30 | HQ .1, Branch B .2 |

WAN links use TEST-NET ranges (RFC 5737) as stand-ins for public IP space, a common convention in lab environments.

## Design Decisions

**Router-on-a-stick at HQ.** A single physical router interface, subdivided into VLAN-specific sub-interfaces, handles inter-VLAN routing for all 3 VLANs. This avoids dedicating a physical interface per VLAN and mirrors how many mid-sized networks are built before a Layer 3 switch upgrade is justified.

**Hub-and-spoke over full mesh.** Branch A and Branch B never tunnel directly to each other — all inter-branch traffic routes through HQ. This centralizes policy enforcement and DHCP/DNS at a single point, at the cost of an extra hop for branch-to-branch traffic (acceptable for this design's scale).

**OSPF instead of static routing.** Routes between all 3 sites are learned dynamically via OSPF (single area, Area 0), rather than manually maintained static routes. This scales cleanly as sites are added and removes a source of manual error.

**GRE tunnels underlying the VPN, rather than plain crypto-map IPsec.** Plain IPsec only encrypts unicast traffic and cannot carry OSPF's multicast hello packets. GRE creates a virtual point-to-point link that OSPF can run over normally; IPsec then encrypts everything flowing through that GRE tunnel. This lets routing stay dynamic even across the encrypted backbone.

**Centralized DHCP/DNS via relay (`ip helper-address`).** Rather than running a DHCP/DNS server at every site, a single server at HQ serves the whole company. Since DHCP requests are local broadcasts that don't cross routers, each site's router relays them to the HQ server as unicast traffic. This is a standard enterprise pattern — fewer servers to patch and manage, consistent DNS records company-wide.

## VPN Encryption — Design Note

Cisco Packet Tracer's simulated IOS does not implement the `crypto isakmp` / IPsec command set, even on a `universalk9`-labeled image (confirmed via `crypto ?`, which lists only `key` as a valid sub-option). As a result, the GRE tunnels in this lab run **unencrypted** in the live topology.

The IPsec configuration below represents the production design — validated for correct Cisco IOS syntax — that would be applied to encrypt each tunnel on real hardware or in a full IOS emulator (e.g. GNS3, EVE-NG):

```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key <shared-key> address <peer-wan-ip>

crypto ipsec transform-set TS-A esp-aes 256 esp-sha256-hmac

access-list 101 permit gre host <local-wan-ip> host <peer-wan-ip>

crypto map CMAP-A 10 ipsec-isakmp
 set peer <peer-wan-ip>
 set transform-set TS-A
 match address 101

interface <physical-wan-interface>
 crypto map CMAP-A
```

Applied identically (with peers/addresses swapped) on both ends of each tunnel, and to each WAN interface — not the tunnel interface itself, since IPsec protects the GRE-encapsulated packet as it crosses the real physical link.

## OSPF

Single area (Area 0) across all site LAN subnets and both tunnel interfaces. The raw public-facing WAN /30 links are intentionally excluded from OSPF — they exist only to establish tunnel reachability, not to carry routed LAN traffic or leak internal topology onto the WAN.

Each site router also carries a static default route toward the ISP router, since this is what makes the GRE tunnel's own destination (the peer's WAN IP) reachable in the first place — OSPF only takes over once a tunnel is already up.

## Troubleshooting Log

A selection of real issues hit during the build, kept here as a demonstration of the debugging process rather than a polished happy-path writeup:

- **Trunk port never actually configured.** After wiring PC-to-switch-to-router at HQ, intra-VLAN pings worked but the router-facing link kept showing up under VLAN 1 (default) in `show vlan brief` — meaning it was never trunked. Root cause: the trunk commands had been run against the wrong interface number. Fixed by reapplying `switchport mode trunk` / `switchport trunk allowed vlan` to the correct port.
- **Cable-to-router landed on an access port.** The switch-to-router cable was physically connected to a port already configured as a VLAN 10 access port (meant for a test PC), rather than the dedicated trunk port — causing the same symptom as above from a different root cause. Fixed by re-cabling to the correct trunk port.
- **Missing routes at the ISP transit router.** Static default routes on each site router fixed outbound reachability, but the ISP router had no return path to either site's LAN subnets — it only knew its own directly-connected /30s. Fixed by adding explicit static routes on the ISP router for each site's LAN summary.
- **Config applied to the wrong device.** More than once, a block of commands intended for a branch router was typed into the ISP router's CLI instead (or vice versa), silently overwriting a working interface's IP address. Caught by comparing `show ip interface brief` output against expected addressing, rather than assuming the config landed correctly. Lesson: confirm the CLI prompt's hostname before pasting any config block.
- **Tunnel source misconfigured as a nonexistent interface.** HQ was mistakenly configured with two separate tunnel sources (`gig0/1` and a nonexistent `gig0/2`), since HQ only has a single physical link to the ISP. Both tunnels correctly share one `tunnel source`; only the `tunnel destination` differs between them.
- **Removing "redundant" static default routes broke the tunnels.** Default routes on the site routers looked redundant once OSPF was running, but they were still required — they're what makes each tunnel's peer WAN IP reachable in the first place, a separate job from OSPF (which only routes traffic *inside* the tunnels). Removing them caused both tunnels to drop, taking OSPF adjacencies down with them. Restored the default routes; only the ISP router's now-redundant static routes to each site's LAN were safe to remove.
- **DHCP relay initially failed — two separate causes.** First, `ip helper-address` was simply missing from a sub-interface (skipped during initial config). Second, after fixing that, a DHCP pool on the server had a typo in its second octet (`10.10.30.1` instead of `10.20.30.1`), pointing the pool at a subnet that didn't exist. Caught by systematically checking router config, server pool config, and reachability separately rather than assuming any one piece was correct.
- **RAM-only config lost on power cycle.** Powering off a router to add a physical module wiped its entire running configuration, since it had never been saved to NVRAM. Reinforced the habit of running `copy running-config startup-config` after any meaningful change, and especially before power-cycling a device.

## Verification Commands

| Purpose | Command |
|---|---|
| Confirm VLAN/port assignment | `show vlan brief` |
| Confirm trunk is active | `show interfaces trunk` |
| Confirm interface/sub-interface state | `show ip interface brief` |
| Confirm routing table contents | `show ip route` |
| Confirm OSPF neighbor adjacency | `show ip ospf neighbor` |
| Confirm CDP-discovered cabling | `show cdp neighbors` |
