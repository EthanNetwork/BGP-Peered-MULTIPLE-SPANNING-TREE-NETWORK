# Enterprise Edge + Campus Access Lab — README

**Platform:** EVEN-NG, IOSv images
**Scope:** Dual-homed BGP edge design, OSPF/HSRP distribution, and a multi-region MST access layer with EtherChannel — built to demonstrate CCNP-level routing and switching skills in one topology.

> **Note on addressing:** All IP addressing in this lab uses public-range-style addresses without PAT/NAT anywhere in the path. In a real production deployment fronting the internet, NAT/PAT would normally sit at the edge. It was intentionally left out here so that routing (BGP/OSPF/HSRP) and switching (MST/EtherChannel) behavior stays fully visible end-to-end without a translation boundary obscuring next-hops, path selection, or reachability testing.

---

## 1. Topology Overview

```
                     isprouter (AS 65002)
                    Gi0/0            Gi0/1
                     |                 |
                     |                 |
           demarc_router_left   demarc_router_right
              (AS 65001)             (AS 65001)
                Gi0/0                  Gi0/0
                     \                 /
                      \               /
                       \             /
                      Switch5 (MST region_1)
                     Gi0/2         Gi0/3
                       |             |
                  Switch4        Switch9 (MST region_3)
              (MST region_2)     Gi0/1,Gi0/2   Gi1/0
                   |               (Po1)        (Po2)
                Switch8              |             |
             (MST region_2)      Switch7        Switch6
                                (MST region_3) (MST region_3)
                                        \          /
                                         \        /
                                        Gi0/2 -- Gi0/2
```

- **isprouter** simulates an upstream ISP, running eBGP to both demarc routers and bridging its two physical links together over a BVI (bridge-group 1 / BVI1) — effectively acting as a single L3 next hop reachable from either demarc router.
- **demarc_router_left** and **demarc_router_right** are the customer edge, dual-homed to the same ISP for redundancy. They run **iBGP** to each other and **eBGP** to the ISP.
- Both demarc routers hand off VLANs 10, 20, and 99 (native) via 802.1Q subinterfaces into the access/distribution layer, with **HSRP** providing a redundant default gateway per VLAN.
- **Switch5** is the distribution switch and sits in its own MST region (`region_1`), acting as root for all instances there.
- **Switch4/Switch8** form one MST region (`region_2`).
- **Switch9/Switch7/Switch6** form a third MST region (`region_3`), interconnected redundantly with two EtherChannel bundles from Switch9 (Po1 → Switch7, Po2 → Switch6) plus a direct link between Switch7 and Switch6 for a full L2 triangle.

---

## 2. Addressing & VLAN Plan

| VLAN | Purpose            | Subnet             | HSRP VIP        |
|------|---------------------|---------------------|------------------|
| 10   | Data VLAN A          | 172.168.2.0/24      | 172.168.2.3      |
| 20   | Data VLAN B          | 172.168.3.0/24      | 172.168.3.3      |
| 99   | Native               | 172.168.1.0/24      | *(no HSRP — routed directly)* |

| Link                          | Subnet               |
|-------------------------------|------------------------|
| demarcROleft ↔ demarcROright ↔ isprouter (BVI) | 201.20.1.0/29 |
| Loopbacks (OSPF router-IDs)   | 10.1.1.1 (isprouter), 10.1.1.253 (left), 10.1.1.254 (right) |

VLANs 10/20 are end-host data VLANs; VLAN 99 doubles as the native VLAN on every trunk in the access/distribution layer **and** as the routed management/transit VLAN between the demarc routers and Switch5.

---

## 3. Routing Design

### 3.1 OSPF (Area 0) — demarc routers only
`demarcROleft` and `demarcROright` run OSPF 100 between their VLAN 10/20/99 subinterfaces and their loopbacks, giving each router a full view of the other's subnets and a router-ID independent of any single interface. `GigabitEthernet0/1` (the link toward the ISP) is passive — OSPF is not permitted to form an adjacency with the ISP router.

### 3.2 HSRP — first-hop redundancy per VLAN
Each data VLAN subinterface runs HSRPv2 with priorities split so that **each demarc router is active for one VLAN and standby for the other** — this spreads the active gateway role across both routers instead of pinning all traffic through one box:

- VLAN 10: `demarcROright` priority 255 (active) / `demarcROleft` priority 254 (standby)
- VLAN 20: `demarcROleft` priority 255 (active) / `demarcROright` priority 254 (standby)

Both sides preempt, so after a failure and recovery the intended active router reclaims its role automatically.

### 3.3 BGP — dual-homed edge, no PAT
- `demarcROleft` and `demarcROright` share **AS 65001** and peer **iBGP** to each other (`next-hop-self` configured so downstream routes aren't lost to unreachable next-hops).
- Both peer **eBGP** to `isprouter` in **AS 65002**, using `ebgp-multihop 2` since the ISP-facing peering IP (201.20.1.3) is reached across a bridged BVI rather than a direct point-to-point link.
- Each demarc router advertises the shared `201.20.1.0/29` transit block via a `network` statement.
- **No PAT/NAT is configured anywhere in this path.** All BGP-advertised and internally routed addressing is used as-is, end-to-end, in a real network PAT would be used, I left it off for the purposes of showcasing specific skills/configs, PAT is used frequently in other repos.

**Verification used:**
```
show ip bgp summary
show ip route bgp
show ip ospf neighbor
show standby brief
```

---

## 4. Switching Design

### 4.1 Trunking
Every inter-switch and switch-to-router link is an 802.1Q trunk carrying VLANs 10, 20, and 99, with **VLAN 99 set as the native VLAN** on every trunk in the topology. Native VLAN is kept consistent end-to-end specifically to avoid native VLAN mismatches (see Section 6).

### 4.2 MST (Multiple Spanning Tree) — three regions
Three independent MST regions were deliberately built to practice region boundary behavior:

| Region     | Members            | Instance 0 (IST)      | MSTI 1 / 10 / 20        |
|------------|----------------------|--------------------------|----------------------------|
| `region_1` | Switch5              | root for all instances (priority 0) | VLAN 10 → MSTI10, VLAN 20 → MSTI20, own instances |
| `region_2` | Switch4, Switch8      | Switch4 root (priority 4096) | VLAN10/20 → MSTI1, Switch4 root (priority 0) |
| `region_3` | Switch9, Switch7, Switch6 | Switch9 root (priority 61440 — tuned as regional root candidate) | VLAN10/20 → MSTI1, Switch9 root (priority 0/4096) |

Since each region has a different name/revision/VLAN-mapping digest, region boundaries fall back to CIST-only (Instance 0) information — MSTI-specific tuning on one region has no visibility or effect on another region's root election, by design.

**Region_3 internal load balancing:** Switch9 connects to both Switch7 and Switch6 via separate EtherChannels (Po1, Po2), and Switch7↔Switch6 have a direct link. MSTI1 cost was intentionally tuned on the Switch7↔Switch6 link (`spanning-tree mst 1 cost 40000`, `20000` on the Switch6 side) to force alternate port forming on the redundant triangle link rather than on either of the primary uplinks to Switch9 and provide failover if an etherchannel goes down.

**Verification used:**
```
show spanning-tree mst configuration
show spanning-tree mst 0
show spanning-tree mst 1
show spanning-tree mst
```

### 4.3 EtherChannel
Switch9 bundles two physical links to Switch7 (Po1) and two physical links to Switch6 (Po2), using static `channel-group <n> mode on`. All trunk parameters (allowed VLANs, native VLAN, encapsulation) are configured on the **port-channel interface** and inherited by the bundled members, rather than configured per-member, to avoid a config mismatch between members.

**Verification used:**
```
show etherchannel summary
show interfaces trunk
show interfaces status err-disabled
```

---

## 5. Skills Demonstrated

- eBGP/iBGP dual-homing to a single upstream AS, with `ebgp-multihop` and `next-hop-self`
- OSPF area design supporting HSRP-fronted subnets
- HSRP active/standby load-splitting across two edge routers
- 802.1Q trunking with a dedicated (non-VLAN-1) native VLAN
- Multiple MST regions with distinct region identities, and reasoning about CIST vs. regional root scope
- Per-instance MST cost/priority tuning to influence root election and port roles within a region
- Static EtherChannel bundling with port-channel-level configuration inheritance
- Structured troubleshooting of native VLAN mismatches, VLAN-database gaps, and EtherChannel misconfig errors

---

## 6. Troubleshooting Log (notable issues hit and resolved)

1. **VLANs not created in local VLAN database on several switches** — trunks showed `trunking` and CDP was healthy, but `show interfaces trunk` reported `none` under "allowed and active," meaning STP never processed BPDUs for VLANs 10/20/99 across those links. Root cause: `switchport trunk allowed vlan` only *permits* VLANs on the trunk, it doesn't create them. Fixed by adding `vlan 10 / vlan 20 / vlan 99` in the VLAN database on every switch — this was also the cause of two switches simultaneously believing they were CIST root, since no BPDUs were actually crossing the affected trunk.

2. **Native VLAN mismatch (switch9 ↔ switch6)** — one side set to native VLAN 99, the other still on default VLAN 1, producing `%CDP-4-NATIVE_VLAN_MISMATCH` and a transient `RECV_PVID_ERR` / `BLOCK_PVID_LOCAL` inconsistency on the port-channel. Resolved by aligning `switchport trunk native vlan 99` on both ends.

3. **EtherChannel `channel-misconfig (STP)` err-disable** — traced through channel-group mode, trunk settings, and MST region alignment (all of which turned out to match) before isolating it to a stale err-disabled state left over from earlier misconfiguration during the session; cleared by bouncing the member interfaces once the underlying causes above were fixed.
# Enterprise Edge + Campus Access Lab — README

**Platform:** EVEN-NG, IOSv images
**Scope:** Dual-homed BGP edge design, OSPF/HSRP distribution, and a multi-region MST access layer with EtherChannel — built to demonstrate CCNP-level routing and switching skills in one topology.

> **Note on addressing:** All IP addressing in this lab uses public-range-style addresses without PAT/NAT anywhere in the path. In a real production deployment fronting the internet, NAT/PAT would normally sit at the edge. It was intentionally left out here so that routing (BGP/OSPF/HSRP) and switching (MST/EtherChannel) behavior stays fully visible end-to-end without a translation boundary obscuring next-hops, path selection, or reachability testing.

---

## 1. Topology Overview

```
                     isprouter (AS 65002)
                    Gi0/0            Gi0/1
                     |                 |
                     |                 |
           demarc_router_left   demarc_router_right
              (AS 65001)             (AS 65001)
                Gi0/0                  Gi0/0
                     \                 /
                      \               /
                       \             /
                      Switch5 (MST region_1)
                     Gi0/2         Gi0/3
                       |             |
                  Switch4        Switch9 (MST region_3)
              (MST region_2)     Gi0/1,Gi0/2   Gi1/0
                   |               (Po1)        (Po2)
                Switch8              |             |
             (MST region_2)      Switch7        Switch6
                                (MST region_3) (MST region_3)
                                        \          /
                                         \        /
                                        Gi0/2 -- Gi0/2
```

- **isprouter** simulates an upstream ISP, running eBGP to both demarc routers and bridging its two physical links together over a BVI (bridge-group 1 / BVI1) — effectively acting as a single L3 next hop reachable from either demarc router.
- **demarc_router_left** and **demarc_router_right** are the customer edge, dual-homed to the same ISP for redundancy. They run **iBGP** to each other and **eBGP** to the ISP.
- Both demarc routers hand off VLANs 10, 20, and 99 (native) via 802.1Q subinterfaces into the access/distribution layer, with **HSRP** providing a redundant default gateway per VLAN.
- **Switch5** is the distribution switch and sits in its own MST region (`region_1`), acting as root for all instances there.
- **Switch4/Switch8** form one MST region (`region_2`).
- **Switch9/Switch7/Switch6** form a third MST region (`region_3`), interconnected redundantly with two EtherChannel bundles from Switch9 (Po1 → Switch7, Po2 → Switch6) plus a direct link between Switch7 and Switch6 for a full L2 triangle.

---

## 2. Addressing & VLAN Plan

| VLAN | Purpose            | Subnet             | HSRP VIP        |
|------|---------------------|---------------------|------------------|
| 10   | Data VLAN A          | 172.168.2.0/24      | 172.168.2.3      |
| 20   | Data VLAN B          | 172.168.3.0/24      | 172.168.3.3      |
| 99   | Native               | 172.168.1.0/24      | *(no HSRP — routed directly)* |

| Link                          | Subnet               |
|-------------------------------|------------------------|
| demarcROleft ↔ demarcROright ↔ isprouter (BVI) | 201.20.1.0/29 |
| Loopbacks (OSPF router-IDs)   | 10.1.1.1 (isprouter), 10.1.1.253 (left), 10.1.1.254 (right) |

VLANs 10/20 are end-host data VLANs; VLAN 99 doubles as the native VLAN on every trunk in the access/distribution layer **and** as the routed management/transit VLAN between the demarc routers and Switch5.

---

## 3. Routing Design

### 3.1 OSPF (Area 0) — demarc routers only
`demarcROleft` and `demarcROright` run OSPF 100 between their VLAN 10/20/99 subinterfaces and their loopbacks, giving each router a full view of the other's subnets and a router-ID independent of any single interface. `GigabitEthernet0/1` (the link toward the ISP) is passive — OSPF is not permitted to form an adjacency with the ISP router.

### 3.2 HSRP — first-hop redundancy per VLAN
Each data VLAN subinterface runs HSRPv2 with priorities split so that **each demarc router is active for one VLAN and standby for the other** — this spreads the active gateway role across both routers instead of pinning all traffic through one box:

- VLAN 10: `demarcROright` priority 255 (active) / `demarcROleft` priority 254 (standby)
- VLAN 20: `demarcROleft` priority 255 (active) / `demarcROright` priority 254 (standby)

Both sides preempt, so after a failure and recovery the intended active router reclaims its role automatically.

### 3.3 BGP — dual-homed edge, no PAT
- `demarcROleft` and `demarcROright` share **AS 65001** and peer **iBGP** to each other (`next-hop-self` configured so downstream routes aren't lost to unreachable next-hops).
- Both peer **eBGP** to `isprouter` in **AS 65002**, using `ebgp-multihop 2` since the ISP-facing peering IP (201.20.1.3) is reached across a bridged BVI rather than a direct point-to-point link.
- Each demarc router advertises the shared `201.20.1.0/29` transit block via a `network` statement.
- **No PAT/NAT is configured anywhere in this path.** All BGP-advertised and internally routed addressing is used as-is, end-to-end, in a real network PAT would be used, I left it off for the purposes of showcasing specific skills/configs, PAT is used frequently in other repos.

**Verification used:**
```
show ip bgp summary
show ip route bgp
show ip ospf neighbor
show standby brief
```

---

## 4. Switching Design

### 4.1 Trunking
Every inter-switch and switch-to-router link is an 802.1Q trunk carrying VLANs 10, 20, and 99, with **VLAN 99 set as the native VLAN** on every trunk in the topology. Native VLAN is kept consistent end-to-end specifically to avoid native VLAN mismatches (see Section 6).

### 4.2 MST (Multiple Spanning Tree) — three regions
Three independent MST regions were deliberately built to practice region boundary behavior:

| Region     | Members            | Instance 0 (IST)      | MSTI 1 / 10 / 20        |
|------------|----------------------|--------------------------|----------------------------|
| `region_1` | Switch5              | root for all instances (priority 0) | VLAN 10 → MSTI10, VLAN 20 → MSTI20, own instances |
| `region_2` | Switch4, Switch8      | Switch4 root (priority 4096) | VLAN10/20 → MSTI1, Switch4 root (priority 0) |
| `region_3` | Switch9, Switch7, Switch6 | Switch9 root (priority 61440 — tuned as regional root candidate) | VLAN10/20 → MSTI1, Switch9 root (priority 0/4096) |

Since each region has a different name/revision/VLAN-mapping digest, region boundaries fall back to CIST-only (Instance 0) information — MSTI-specific tuning on one region has no visibility or effect on another region's root election, by design.

**Region_3 internal load balancing:** Switch9 connects to both Switch7 and Switch6 via separate EtherChannels (Po1, Po2), and Switch7↔Switch6 have a direct link. MSTI1 cost was intentionally tuned on the Switch7↔Switch6 link (`spanning-tree mst 1 cost 40000`, `20000` on the Switch6 side) to force alternate port forming on the redundant triangle link rather than on either of the primary uplinks to Switch9 and provide failover if an etherchannel goes down.

**Verification used:**
```
show spanning-tree mst configuration
show spanning-tree mst 0
show spanning-tree mst 1
show spanning-tree mst
```

### 4.3 EtherChannel
Switch9 bundles two physical links to Switch7 (Po1) and two physical links to Switch6 (Po2), using static `channel-group <n> mode on`. All trunk parameters (allowed VLANs, native VLAN, encapsulation) are configured on the **port-channel interface** and inherited by the bundled members, rather than configured per-member, to avoid a config mismatch between members.

**Verification used:**
```
show etherchannel summary
show interfaces trunk
show interfaces status err-disabled
```

---

## 5. Skills Demonstrated

- eBGP/iBGP dual-homing to a single upstream AS, with `ebgp-multihop` and `next-hop-self`
- OSPF area design supporting HSRP-fronted subnets
- HSRP active/standby load-splitting across two edge routers
- 802.1Q trunking with a dedicated (non-VLAN-1) native VLAN
- Multiple MST regions with distinct region identities, and reasoning about CIST vs. regional root scope
- Per-instance MST cost/priority tuning to influence root election and port roles within a region
- Static EtherChannel bundling with port-channel-level configuration inheritance
- Structured troubleshooting of native VLAN mismatches, VLAN-database gaps, and EtherChannel misconfig errors

---

## 6. Troubleshooting Log (notable issues hit and resolved)

1. **VLANs not created in local VLAN database on several switches** — trunks showed `trunking` and CDP was healthy, but `show interfaces trunk` reported `none` under "allowed and active," meaning STP never processed BPDUs for VLANs 10/20/99 across those links. Root cause: `switchport trunk allowed vlan` only *permits* VLANs on the trunk, it doesn't create them. Fixed by adding `vlan 10 / vlan 20 / vlan 99` in the VLAN database on every switch — this was also the cause of two switches simultaneously believing they were CIST root, since no BPDUs were actually crossing the affected trunk.

2. **Native VLAN mismatch (switch9 ↔ switch6)** — one side set to native VLAN 99, the other still on default VLAN 1, producing `%CDP-4-NATIVE_VLAN_MISMATCH` and a transient `RECV_PVID_ERR` / `BLOCK_PVID_LOCAL` inconsistency on the port-channel. Resolved by aligning `switchport trunk native vlan 99` on both ends.

3. **EtherChannel `channel-misconfig (STP)` err-disable** — traced through channel-group mode, trunk settings, and MST region alignment (all of which turned out to match) before isolating it to a stale err-disabled state left over from earlier misconfiguration during the session; cleared by bouncing the member interfaces once the underlying causes above were fixed.
